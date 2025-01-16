+++
date = '2025-01-15T15:59:04-07:00'
draft = false
title = 'Private NixOS Configurations'
+++


# Overview

For newcomers to Nix, especially those coming from other 'traditional' distros, the documentation can be quite challenging. Nix in itself is a programming language and requires a baseline of knowledge to understand what it's doing.

If you want to learn about general Linux things like specific packages or services, how they are configured, etc, the [ArchWiki](https://wiki.archlinux.org/title/Arch_Linux) is, in my opinion, bar none the best resource. Want to learn about Nix and NixOS, [Nix.dev](https://nix.dev/), [NixPkgs](https://github.com/NixOS/nixpkgs) and public user configurations have been my go to. While there are some "best practices," it is still a programming language which inherently allows for different approaches and solutions.

This is especially true in regards to personal repository structure and configurations. Because of this, when I first began my Nix journey one of my favorite things to do was explore other people's public configurations. This is the reason I personally wanted to make my own configuration public. You just never know, your configuration may be the one that makes it "click" or provide solutions to someones problem or even just be flat out inspirational.

I know I was personally inspired by [Jake Hamilton's](https://github.com/jakehamilton/config) config. In fact, you'll see his opinionated approach in my current repository structure. But at the time, I just didn't quite understand everything his custom library was doing, so I didn't fully adopt it. As it goes when learning anything new, I wanted to get my hands dirty writing my own Nix code.

You might be thinking, "is it safe to store an entire OS configuration in a PUBLIC repository!? Do I really want all my configurations to be public? What about the sensitive stuff? Like user passwords or API keys? That doesn't sound secure!." Okay, you're probably not thinking that exactly but obviously there is a reason you're reading this post. There are aspects of your NixOS configurations that you want or need to keep private.

In my case, I wanted to migrate away from Ansible managed docker container deployments to a Nix managed deployment. Since some of my containers are public facing, I'd prefer to not expose the applications, their specific versions or the TLDs I use for them. On top of that, a handful of my container configurations require secrets. This is easy to do when using ansible-vault and/or passing them in via Gitlab CI/CD variables, but I needed to Nixify things.

So I had two goals:

 - Create NixOS configurations that are stored in a private repository and utilized in my public repository.
 - Create and manage secrets that are stored as an encrypted file in a private repository that can also be utilized in my public repository.

Fortunately, the inherit functionality of [Nix Flakes](https://wiki.nixos.org/wiki/Flakes) makes the incorporation of private repositories pretty straightforward solving the first goal. As for the second goal, as most things with Nix, someone has thought of and provided a solution for declaratively configuring and managing secrets within NixOS configurations by creating [sops-nix](https://github.com/Mic92/sops-nix).

This guide will provide a walkthrough on how I personally accomplished creating private container configs and the workflow I established for a somewhat frictionless experience. A second guide will expand on this configuration and demonstrate how I manage secrets using sops-nix and packaging the encrypted files as a Nix package which can be used in both the public and private configurations.

# But before we begin...

There are a fair amount of assumptions that will be made in these guides. First, that you are utilizing flakes with your NixOS config and also have general familiarity with the Nix language. I intend for these guides to be pretty hand-holdy and provide as much detail as possible while hopefully not bogging down those are already familiar with subject matter.

I also want to point out one of the more obvious caveats to using a private repository within your flake which is the requirement of providing authentication to the nix cli when performing builds or `flake.lock` updates. Without authentication, it will error that the repository is not readable and fall back to the most recent cached version. In my case, since my private repository is on GitHub, I will need to create a [GitHub Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) that is scoped with appropriate repository Read Access. This configuration will be covered in a later section of the guide and I will demonstrate the changes I have incorporated into my workflow.

A working example can be found in my [public nixos-config repository](https://github.com/thursdaddy/nixos-config).
# Private Flake.nix
 First we need to create a private repository for our `flake.nix`. I am going to call mine `nixos-thurs`.

```bash
$ git init nixos-thurs
```

Create a boilerplate `flake.nix`.

```nix
{
  description = "For my eyes only.";

  inputs.nixpkgs.url = "nixpkgs";

  outputs = { self, nixpkgs, ... } @ inputs:
  {
    packages = { };
    nixosModules = { };
  };
}

```

For demonstration purposes, I'm going to go ahead and commit and push this `flake.nix` as is to a private GitHub repo.

In your main `flake.nix` add the private repo as an input.

>Don't forget to add the name of the input to the `outputs = { }` section.

```nix
{
  description = "Nix the planet!";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-24.11";

    # private repo
    nixos-thurs = {
      url = "github:thursdaddy/nixos-thurs/main";
    };
  };

  outputs = { self, nixpkgs, nixos-thurs, ... } @ inputs:
    { ... ommitted for brevity

```

Let's try it out.
```bash
$ nix flake update
error:
       … while updating the lock file of flake 'git+file:///your/nixos/config/path'
       … while updating the flake input 'nixos-thurs'
       … while fetching the input 'github:thursdaddy/nixos-thurs/main'
       error: unable to download 'https://api.github.com/repos/thursdaddy/nixos-thurs/commits/main': HTTP error 404
```

As you can see, without authentication `nix flake update` will fail to clone your repository and the only method available via the nix cli is in the form of [access tokens](https://nix.dev/manual/nix/2.17/command-ref/conf-file#conf-access-tokens).

According to nix.conf [documentation](https://nix.dev/manual/nix/2.17/command-ref/conf-file#configuration-file) we can configure these via environment variables or within the `nix.conf` file.

Let's go ahead and temporarily set our access token by setting the `NIX_CONFIG` environment variable when we run `nix flake update`

> Tip: Most default bash/zsh terminal configurations will prevent a command from being saved in history if it begins with a space.

```bash
$  NIX_CONFIG="access-tokens = github.com=github_pat_...." nix flake update
warning: creating lock file '/your/nixos/config/path/flake.lock':
• Added input 'nixpkgs':
    'github:nixos/nixpkgs/9c6b49aeac36e2ed73a8c472f1546f6d9cf1addc?narHash=sha256-i/UJ5I7HoqmFMwZEH6vAvBxOrjjOJNU739lnZnhUln8%3D' (2025-01-14)
• Added input 'nixos-thurs':
    'github:thursdaddy/nixos-thurs/023c88ac041b5793c2648d692ecfd1c518915781?narHash=sha256-zSZ2v2IgfNE81aFNpJBUkdj4TsyVh53MnnDJ5eH8ZZk%3D' (2025-01-15)
• Added input 'nixos-thurs/nixpkgs':
    'path:/nix/store/mr3pq6jnd0j7vny01q8r8a2ij8z2mn2d-source?lastModified=1735669367&narHash=sha256-tfYRbFhMOnYaM4ippqqid3BaLOXoFNdImrfBfCp4zn0%3D&rev=edf04b75c13c2ac0e54df5ec5c543e300f76f1c9' (2024-12-31)
```

Boom, now we're cookin.

# Container Configurations

Depending on your level of NixFU, you may or may not be aware of the concept that Nix flakes, in simple terms, are [input -> output processors of nix code](https://zero-to-nix.com/concepts/flakes/).

With this concept in mind, we are going to create a custom [nixosModule](https://nixos.wiki/wiki/NixOS_modules) in the `outputs` section of our private `flake.nix`. This output will define our container configurations and because our private repo is an input in our main `flake.nix` the configuration can be imported via `nixosConfigurations.<name>.nixosSystem.modules`. Hopefully that makes sense but stay with me even if it doesn't...

Back in our private `flake.nix` we are going to create `nixosModules.myContainers`:
```nix
{
  description = "For my eyes only.";

  inputs.nixpkgs.url = "nixpkgs";

  outputs = { self, nixpkgs, ... } @ inputs:
  {
    packages = { };
    nixosModules.myContainers = { pkgs, ... }: {
      config = {
        virtualisation.oci-containers.containers."nginx" = {
          image = "nginx";
          ports = [ "80:80" ];
        };
      };
    };
  };
}
```

Since I previously pushed my boilerplate private `flake.nix` I need to commit and push these changes and of course, update my main `flake.lock`.
```bash
$  NIX_CONFIG="access-tokens = github.com=github_pat_...." nix flake update
warning: updating lock file '/your/nixos/config/path/flake.lock':
• Updated input 'nixos-thurs':
    'github:thursdaddy/nixos-thurs/023c88ac041b5793c2648d692ecfd1c518915781?narHash=sha256-zSZ2v2IgfNE81aFNpJBUkdj4TsyVh53MnnDJ5eH8ZZk%3D' (2025-01-15)
  → 'github:thursdaddy/nixos-thurs/7c0e000de42a58e45abedcd130a4d07c5a3876ee?narHash=sha256-RASNNil5UlyT%2BQ7SvGm5BQzwxVCM1jJ16QSZHlq2ClY%3D' (2025-01-16)
```
Now, lets use our input to import `nixosModule.myContainers` via `nixosConfigurations.<name>.nixosSystem.modules`.

For this example, I'll do so under my `c137` configuration:
```nix
{
  description = "Nix the planet!";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-24.11";

    # private repo
    nixos-thurs = {
      url = "github:thursdaddy/nixos-thurs/main";
    };
  };

  outputs = { self, nixpkgs, nixos-thurs, ... } @ inputs: {
    nixosConfigurations = {
      "c137" = nixpkgs.lib.nixosSystem {
        specialArgs = { inherit inputs; };
        system = "x86_64-linux";
        modules = [
          ./hosts/c137/configuration.nix
          inputs.nixos-thurs.nixosModules.myContainers # <--------- HERE
        ];
      };
    };
  };
}
```
When you perform your next `nixos-rebuild` you should see the systemctl unit starting the docker container. Easy as that! Now, obviously, this functionality is not limited to just docker container configurations and should be viable for any configurations that you'd like to keep private. This can even be expanded to creating packages, which will be covered in another guide.

In the next section, I'll cover the workflow and some helper tools I am using to improve my testing and development experience. If you have a good understanding of the requirements and your own methods of madness then I hope this guide has helped expand your knowledge on the utility Nix flakes provide.

# My personal workflow
You may have picked up on some less-than-ideal processes when dealing with a private input in your `flake.nix`. For starters, having to push your changes to your private repo, then update your `flake.lock` to pull down those new changes does not sounds like a fun time.

What if you just want to do some quick testing? Rather not fill up your private reposistories history with `"fix: typo"` commits? Me too!

Conveniently, the Nix flake spec allows you to specify the [local path](https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-flake#path-like-syntax) to your private `flake.nix`.

```bash
    # private nixos configs
    nixos-thurs = {
      url = "git+file:///your/nixos/config/path/nixos-thurs/";
    };

```
> Flakes are married to git, you will need stage your files in the private repository or they will not be included when imported into your main `flake.nix`

This makes local development pretty straight-forward. Make changes in your private repository, `git add` them and they will automatically be pulled in with the next `nixos-rebuild` command in your main `flake.nix`, no need to `nix flake update` beforehand.

For some people, setting a local path to your private repository may be perfectly acceptable as long term solution. Doing so actually avoids some of the hassles with having to authenticate to a private repository which is one less hurdle to deal with. But then you lose a form of remote backup, or you may start getting lazy with your commits and things start getting messy. I'd rather not.

To get the best of both worlds, quick and easy local development as well as a remote backup and enforcing better git practices, I went the route of writing a little bash wrapper script to help manage things.

### BEWARE, HERE BE DRAGONS
 The full script can be found in my public [nixos-config](https://github.com/thursdaddy/nixos-config/blob/main/nix.sh) repository and I am just going to cover some of the functions that solve the workflow issues mentioned previously.

 Please, don't judge too harshly, some of these functions are rough, to say the least, but they work! As these are unique to my use case, I am just highlighting them as they may be a source of inspiration when developing your own workflow.

> I use a Justfile with this wrapper script to help streamline commands and arguments. If you're not familiar, check it out, it's like Makefile but simpler.

Setting GitHub authorization token, if it exists:
```bash
check_gh_token () {
  # set GH_TOKEN to pull private flake from private repo
  if [ -f "/run/secrets/github/TOKEN" ] || [ -f "$HOME/.config/sops-nix/secrets/github/TOKEN" ]; then
    if [[ ${HOSTNAME:-$HOST} =~ "mbp" ]]; then
      printf "${ORANGE}GitHub token found!${NC}"
      GH_TOKEN=$(cat ~/.config/sops-nix/secrets/github/TOKEN)
    else
      printf "${ORANGE}GitHub token found!${NC}"
      GH_TOKEN=$(cat /run/secrets/github/TOKEN)
    fi
    export NIX_CONFIG="extra-access-tokens = github.com=${GH_TOKEN}"
  else
    printf "${ORANGE}No GH token set.${NC}"
    GH_TOKEN=''
  fi
}
```
Updating from public to private inputs for `nixos-thurs`:
```bash
update_to_local_input () {
  # set path based on target
  printf "${BLUE}Setting input to local: ${WHITE}${INPUT}${NC}\n"
  if [[ ${HOSTNAME:-$HOST} =~ "mbp" ]]; then
    local NIXOS_THURS_PATH="\/Users\/thurs"
  else
    local NIXOS_THURS_PATH="\/home\/thurs"
  fi
  # if github url is set, replace it
  sed -i 's/      url = ".*'"${INPUT}"'.*/      url = "git+file:\/\/'"${NIXOS_THURS_PATH}"'\/projects\/nix\/'"${INPUT}"'\/";/g' flake.nix

  # finally update
  nix flake update "${INPUT}"
}
```
Example:
```bash
$ just local nixos-thurs
Setting input to local: nixos-thurs
warning: updating lock file '/your/nixos/config/path/nixos-config/flake.lock':
• Updated input 'nixos-thurs':
    'github:thursdaddy/nixos-thurs/a4c71606282f27c441070b1dc9c94040687fc030?narHash=sha256-hPeBuYXISo3RPPE9s6jCwnXVoBhz7bgKgbiqIV7Ph2I%3D' (2025-01-16)
  → 'git+file:///your/nixos/config/path/nixos-thurs/?ref=refs/heads/main&rev=a4c71606282f27c441070b1dc9c94040687fc030' (2025-01-16)

```
Updating from private to public inputs for `nixos-thurs`:
```bash
update_flake_input () {
  check_gh_token
  if [[ $INPUT == "nixos-thurs" ]]; then
    sed -i 's/      url = ".*'"${INPUT}"'.*/      url = "github:thursdaddy\/'"${INPUT}"'\/main";/g' flake.nix
  fi

  if [ "${INPUT}" == "all" ]; then
    printf "\n${GREEN}Updating flake.nix...${NC}\n"
    nix flake update
  else
    printf "\n${GREEN}Updating flake.nix input: ${WHITE}${INPUT}${NC}\n"
    nix flake update "${INPUT}"
  fi
}
```
Example:
```bash
$ just update nixos-thurs
GitHub token found!
Updating flake.nix input: nixos-thurs
warning: updating lock file '/your/nixos/config/path/nixos-config/flake.lock':
• Updated input 'nixos-thurs':
    'github:thursdaddy/nixos-thurs/557bf321f1d315871d70a0c46c66b5f733228e7c?narHash=sha256-M8tO42FuMqSJBLzli9IjqyCfqC86155ZhoBBs8/giN0%3D' (2025-01-08)
  → 'github:thursdaddy/nixos-thurs/a4c71606282f27c441070b1dc9c94040687fc030?narHash=sha256-hPeBuYXISo3RPPE9s6jCwnXVoBhz7bgKgbiqIV7Ph2I%3D' (2025-01-16)
```

# Conclusion

I think I've covered just about everything related to setting up and working with a private repository input in your `flake.nix`. Hopefully it was helpful! Feel free to open an [issue](https://github.com/thursdaddy/nixos-config/issues) on my public nixos-config repository if you have any questions or would like clarification on anything.

Stay tuned to the Part 2 of this guide which will cover sops, sop-nix and how to package your encrypted secrets files as a "private" Nix pacakge!
