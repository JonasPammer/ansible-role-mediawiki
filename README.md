<!-- This file is being generated by .github/workflows/gh-pages.yml - all local changes will be lost eventually! -->
[![Version on Galaxy](https://img.shields.io/badge/available%20on%20ansible%20galaxy-jonaspammer.mediawiki-brightgreen)](https://galaxy.ansible.com/jonaspammer/mediawiki) [![Testing CI](https://github.com/JonasPammer/ansible-role-mediawiki/actions/workflows/ci.yml/badge.svg)](https://github.com/JonasPammer/ansible-role-mediawiki/actions/workflows/ci.yml)

An Ansible role for

- downloading mediawiki into a specified directory,

- installing recommended system packages

- downloading additional extensions and skins through Git or Composer

This role does **not** install/configure an SQL-Server.

This role does **not** install/configure a Webserver.

This role does **not** generate a LocalSettings.php ([yet](https://github.com/JonasPammer/ansible-role-mediawiki/issues/2)).

This role does **not** execute any of Mediawiki’s maintenance scripts (like `update.php` which you’ll want to run for most extensions to work) [yet](https://github.com/JonasPammer/ansible-role-mediawiki/issues/3).

This role **does** try to ensure permissions of Mediawiki’s files.

# 🔎 Metadata

Below you can find information on…

-   the role’s required Ansible version

-   the role’s supported platforms

-   the role’s [role dependencies](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#role-dependencies)

**[meta/main.yml](meta/main.yml)**

    ---
    galaxy_info:
      role_name: "mediawiki"
      description:
        "An ansible role for downloading mediawiki into specified directory,
        installing recommended system packages and downloading additional extensions and skins."

      author: "jonaspammer"
      license: "MIT"

      min_ansible_version: "2.11"
      platforms:
        # note: text after "actively tested: " represent the docker image name
        - name: EL # (Enterprise Linux)
          versions:
            - "9" # actively tested: rockylinux9
        - name: Fedora
          versions:
            - "38" # actively tested: fedora38
            - "39" # actively tested: fedora39
        - name: Debian
          versions:
            - bullseye # actively tested: debian11
            - bookworm # actively tested: debian12
        - name: Ubuntu
          versions:
            - focal # actively tested: ubuntu2004
            - jammy # actively tested: ubuntu2204

      galaxy_tags: []

    dependencies: []

# 📌 Requirements

The Ansible User needs to be able to `become`.

The [`community.general` collection](https://galaxy.ansible.com/community/general) must be installed on the Ansible controller.

Composer needs to be installed on the Host.

# 📜 Role Variables

    # php_version: "7.2"

**Required to be set by yourself.** Used in the names of the installed PHP Packages. Variable-Name comes from the `php-versions` role.

Keep in mind that the php version satisfies [MediaWiki’s Compatibility Matrix](https://www.mediawiki.org/wiki/Compatibility).

This variable name intentionally overlaps with [ansible-role-php-versions](https://github.com/geerlingguy/ansible-role-php-versions). It’s recommended to have it be set in your host_vars specification, because if you accidentally install even just one php package using an alternating version specifier **it can mess up your entire system** (PHP system packages are weird).

Leaving this blank will result in the installation of the symlink-packages of the system.

    mediawiki_version_major: 1
    mediawiki_version_minor: 36
    mediawiki_version_release: 0

Version of Mediawiki to download. See [releases.wikimedia.org](https://releases.wikimedia.org/mediawiki/) and [Wikipedia: MediaWiki Version History](https://en.wikipedia.org/wiki/MediaWiki_version_history).

    mediawiki_destination: [default htdocs of distro]/mediawiki

Folder in which to extract the downloaded mediawiki archive into. Must **not** end with an "/".

The default points to a directory named `mediawiki` in the default httpd' directory of the distribution:

- default: `/var/www/html`

- Alpine: `/var/www/{{ httpd_servername | default(ansible_fqdn) }}`

- Suse: `/srv/www/htdocs`

<!-- -->

    mediawiki_destination_permissions: u=rwx,g=rx,o=rx

The permissions the resulting directory should have.

    mediawiki_linux_username: ~
    mediawiki_linux_group: ~

User and Group that should own the destination directory itself.

You will need to ensure these are created **beforehand** (e.g. using `pre_tasks`) - the machine’s passwd configuration is no business to this role.

    mediawiki_enable_webserver: true

Enable/Disable this role’s `restart httpd` handler.

## Downloading Extensions

This can be skipped by [ skipping the tag](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#selecting-or-skipping-tags-when-you-run-a-playbook) `mediawiki::extensions`.

By using this role in combination with this variable and its structure you could have something like this in your LocalSettings.php Template:

    {% for category_identfier, extensions_array in mediawiki_extensions.items() %}
    # {{ category_identfier }} extensions
    {%  for extension in extensions_array %}
    {%      if extension.load %}
    wfLoadExtension("{{ extension.name }}");
    {%      endif %}
    {%  endfor %}
    {% endfor %}

    mediawiki_extensions:
      unsorted: []

A _dictionary of lists_ of Extensions to download.

The dictionary keys are to attach an _arbitrary_ "category" to each extension. How you name these "categories" is to your business only.

Each entry of a list may have the following properties (Consult the [📚 Example Playbook Usages](#example_playbooks)-Section for Examples):

name
Name of the Extension as used when registering the extension in `LocalSettings.php`.

load
Boolean. Can be used in the Jinja2 Template to decide if the extension shall be loaded. Does not have any effect in this role.

gather_type
This variable defines how to gather the extension. Possible values: "composer", "git". Defaults to "git".

Extensions can be **gathered** for a given MediaWiki-Version through various ways. As of 2021, the most common/supported way is by…

- Downloading the extension from Git to the `/extensions`-directory

- Optionally running `composer install [--no-dev …]` in the cloned directory to install its dependencies in _its_ directory (kind-of-like `npm install`).

  When you download an extension from [ MediaWiki’s Extension distributor](https://www.mediawiki.org/wiki/Special:ExtensionDistributor), this step has already been done beforehand.

A more recent initiative attempts to implement the **sole** use of Composer to gather Mediawiki’s Extensions (instead of just using it for gathering libraries), for-example by issuing `composer require mediawiki/semantic-media-wiki` in Mediawiki’s base directory. This is still [an actively discussed RFC](https://phabricator.wikimedia.org/T250406).

This method can only be done if the extension exists as a "Composer package" of-course.

No-matter which version is used to gather the extension, you’ll still need to issue `wfLoadExtension` in your "LocalSettings.php"-file.

composer_name
Name of the composer package of the Extension, for example as found on [ packagist.org](https://packagist.org/search/?type=mediawiki-extension).

It’s a good Idea to pass in this value even if you plan to use git as the gather-method, assuming your Extensions [ exists as a composer package](https://www.mediawiki.org/wiki/Category:Extensions_supporting_Composer). By doing so, this role can make sure Mediawiki’s Composer does not contain this Composer Package (which could cause the weirdest conflicts).

Also, if you do this, I like to explicitly specify the `gather_type` to be "git" myself.

composer_version
[Version Constraint](https://getcomposer.org/doc/articles/versions.md#writing-version-constraints) for the Composer Package.

_git_mwrepo_name_
If your extensions is under [ Wikimedias' version control](https://www.mediawiki.org/wiki/Category:Extensions_in_Wikimedia_version_control), but uses a different name for their Repository than provided in `name`, you can use this to supply the name as used in the MediaWiki Repository. Look at the default of `git_url` to understand this. Defaults to `name`.

git_url
URL to `.git` from the repository of the extension. Defaults to `https://github.com/wikimedia/mediawiki-extensions-{{ git_mwrepo_name }}.git`.

git*version
What version of the repository to check out. This can be the literal string HEAD, a branch name, a tag name. Defaults to `REL{{ mediawiki_version_major }}*{{ mediawiki_version_minor }}` if not provided.

git_run_composer_install
Boolean or "always". Whether to run `composer install` in the directory of the Extension. Defaults to value of `mediawiki_extensions_git_run_composer_install_default`.

- If set to "always", the command will be executed on every run.

- If set to a truthy boolean value, the command will be executed if the issued git module reports a change.

_system_package_dependencies_
Package name(s) to install to the system using [ ansible.builtin.package](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html#parameter-name).

<!-- -->

    mediawiki_extensions_git_run_composer_install_default: true

Overwrites the default value for `git_run_composer_install` of every extension.

## Downloading Skins

This can be skipped by [ skipping the tag](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#selecting-or-skipping-tags-when-you-run-a-playbook) `mediawiki::skins`.

By using this role in combination with this variable and its structure you could have something like this in your LocalSettings.php Template:

    {% for skin in mediawiki_skins %}
    wfLoadSkin( '{{ skin.name }}' );
    {% endfor %}

    mediawiki_skins: []

A list of Skins to download.

Each entry of the list may have the following properties (Consult the [📚 Example Playbook Usages](#example_playbooks)-Section for Examples):

name
Official Name, as used when loading the skin. If your extensions falls under [ Wikimedias' version control](https://www.mediawiki.org/wiki/Category:Extensions_in_Wikimedia_version_control) you will only need to supply this value.

git_url
URL to `.git` from the repository of the extension. Defaults to `https://github.com/wikimedia/mediawiki-extensions-{{ name }}.git` if not provided.

git*version
What version of the repository to check out. This can be the literal string HEAD, a branch name, a tag name. Defaults to `REL{{ mediawiki_version_major }}*{{ mediawiki_version_minor }}` if not provided.

# 📜 Facts/Variables defined by this role

Each variable listed in this section is dynamically defined when executing this role (and can only be overwritten using `ansible.builtin.set_facts`) *and* is meant to be used not just internally.

# 🏷️ Tags

Tasks are tagged with the following [tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#adding-tags-to-roles):

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Tag</th>
<th style="text-align: left;">Purpose</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><p>mediawiki::extensions</p></td>
<td style="text-align: left;"></td>
</tr>
<tr class="even">
<td style="text-align: left;"><p>mediawiki::skins</p></td>
<td style="text-align: left;"></td>
</tr>
</tbody>
</table>

You can use Ansible to skip tasks, or only run certain tasks by using these tags. By default, all tasks are run when no tags are specified.

# 👫 Dependencies

- [geerlingguy.php](https://github.com/geerlingguy/ansible-role-php) (This role only installs packages not included in the defaults of linked role)

- [geerlingguy.php-mysql](https://github.com/geerlingguy/ansible-role-php-mysql)

# 📚 Example Playbook Usages

This role is part of [ many compatible purpose-specific roles of mine](https://github.com/JonasPammer/ansible-roles).

The machine needs to be prepared. In CI, this is done in `molecule/resources/prepare.yml` which sources its soft dependencies from `requirements.yml`:

**[molecule/resources/prepare.yml](molecule/resources/prepare.yml)**

    ---
    - name: prepare
      hosts: all
      become: true
      gather_facts: false

      vars:
        # https://www.mediawiki.org/wiki/Compatibility
        # https://www.php.net/supported-versions.php
        php_version: "7.4"
        php_enable_webserver: false

      roles:
        - role: jonaspammer.bootstrap
        - role: geerlingguy.repo-epel
          # repo-epel is "a collection of selected packages from fedora" and thus not needed for fedora
          when: ansible_os_family == "RedHat" and ansible_distribution != "Fedora"
        - role: geerlingguy.repo-remi
          # repo-remi is only supported on redhat
          when: ansible_os_family == "RedHat"
        - role: geerlingguy.php-versions
        - role: geerlingguy.php
        - role: geerlingguy.php-mysql
        - role: geerlingguy.git
        #    - role: jonaspammer.core_dependencies

The following diagram is a compilation of the "soft dependencies" of this role as well as the recursive tree of their soft dependencies.

![requirements.yml dependency graph of jonaspammer.mediawiki](https://raw.githubusercontent.com/JonasPammer/ansible-roles/master/graphs/dependencies_mediawiki.svg)

    roles:
      - jonaspammer.mediawiki

    vars:
      mediawiki_destination: "/opt/my_wiki"
      mediawiki_linux_username: "root"
      mediawiki_linux_group: "root"

If an extensions is under [ Wikimedias' version control](https://www.mediawiki.org/wiki/Category:Extensions_in_Wikimedia_version_control), you will only need to supply the `name` property.

    roles:
      - geerlingguy.git
      - jonaspammer.mediawiki

    vars:
      mediawiki_extensions:
        special_page:
          - name: "ExtendedFilelist"
            git_mwrepo_name: "BlueSpiceExtendedFilelist"
            git_run_composer_install: true

        editor:
          - name: "CodeEditor"
          - name: "CodeMirror"
          - name: "VisualEditor"
          - name: "WikiEditor"

        parser:
          - name: "BOFH"
            git_url: "https://github.com/tessus/mwExtensionBOFH"
            git_version: "1.8"

        semantic_mediawiki:
          - name: "SemanticMediaWiki"
            gather_type: composer
            composer_name: "mediawiki/semantic-media-wiki"
            composer_version: "~3.0"

        variable:
          - name: "HitCounters"
            gather_type: git  # We get it from git...
            composer_name: "mediawiki/hit-counters"  # ...but make sure that, if it was previously installed through composer, this role removes it from Mediawiki's Composer packages

      mediawiki_skins:
        - name: "Timeless"
        - name: "Vector"
        - name: "MonoBook"
        - name: "MinervaNeue"

# 🧪 Tested Distributions

A role may work on different **distributions**, like Red Hat Enterprise Linux (RHEL), even though there is no test for this exact distribution.

<table>
<colgroup>
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">OS Family</th>
<th style="text-align: left;">Distribution</th>
<th style="text-align: left;">Distribution Release Date</th>
<th style="text-align: left;">Distribution End of Life</th>
<th style="text-align: left;">Accompanying Docker Image</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><p>Rocky</p></td>
<td style="text-align: left;"><p>Rocky Linux 8 (<a href="https://www.howtogeek.com/devops/is-rocky-linux-the-new-centos/">RHEL/CentOS 8 in disguise</a>)</p></td>
<td style="text-align: left;"><p>2021-06</p></td>
<td style="text-align: left;"><p>2029-05</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-rockylinux8-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-rockylinux8-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="even">
<td style="text-align: left;"><p>Rocky</p></td>
<td style="text-align: left;"><p>Rocky Linux 9</p></td>
<td style="text-align: left;"><p>2022-07</p></td>
<td style="text-align: left;"><p>2032-05</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-rockylinux9-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-rockylinux9-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="odd">
<td style="text-align: left;"><p>RedHat</p></td>
<td style="text-align: left;"><p>Fedora 39</p></td>
<td style="text-align: left;"><p>2023-11</p></td>
<td style="text-align: left;"><p>2024-12</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-fedora39-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-fedora39-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="even">
<td style="text-align: left;"><p>Debian</p></td>
<td style="text-align: left;"><p>Ubuntu 20.04 LTS</p></td>
<td style="text-align: left;"><p>2021-04</p></td>
<td style="text-align: left;"><p>2025-04</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-ubuntu2004-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-ubuntu2004-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="odd">
<td style="text-align: left;"><p>Debian</p></td>
<td style="text-align: left;"><p>Ubuntu 22.04 LTS</p></td>
<td style="text-align: left;"><p>2022-04</p></td>
<td style="text-align: left;"><p>2027-04</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-ubuntu2204-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-ubuntu2204-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="even">
<td style="text-align: left;"><p>Debian</p></td>
<td style="text-align: left;"><p>Debian 11</p></td>
<td style="text-align: left;"><p>2021-08</p></td>
<td style="text-align: left;"><p>2024-06 (2026-06 LTS)</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-debian11-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-debian11-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
<tr class="odd">
<td style="text-align: left;"><p>Debian</p></td>
<td style="text-align: left;"><p>Debian 12</p></td>
<td style="text-align: left;"><p>2023-06</p></td>
<td style="text-align: left;"><p>2026-06 (2028-06 LTS)</p></td>
<td style="text-align: left;"><p><a href="https://github.com/geerlingguy/docker-debian12-ansible/actions?query=workflow%3ABuild"><img src="https://github.com/geerlingguy/docker-debian12-ansible/workflows/Build/badge.svg?branch=master" alt="CI" /></a></p></td>
</tr>
</tbody>
</table>

# 🧪 Tested Ansible versions

The tested ansible versions try to stay equivalent with the [ support pattern of Ansible’s `community.general` collection](https://github.com/ansible-collections/community.general#tested-with-ansible). As of writing this is:

-   2.13 (Ansible 6)

-   2.14 (Ansible 7)

-   2.15 (Ansible 8)

-   2.16 (Ansible 9)

# 📝 Development

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-yellow.svg)](https://conventionalcommits.org) [![pre-commit.ci status](https://results.pre-commit.ci/badge/github/JonasPammer/ansible-role-mediawiki/master.svg)](https://results.pre-commit.ci/latest/github/JonasPammer/ansible-role-mediawiki/master)

## 📌 Development Machine Dependencies

-   Python 3.10 or greater

-   Docker

## 📌 Development Dependencies

Development Dependencies are defined in a [pip requirements file](https://pip.pypa.io/en/stable/user_guide/#requirements-files) named `requirements-dev.txt`. Example Installation Instructions for Linux are shown below:

    # "optional": create a python virtualenv and activate it for the current shell session
    $ python3 -m venv venv
    $ source venv/bin/activate

    $ python3 -m pip install -r requirements-dev.txt

## ℹ️ Ansible Role Development Guidelines

Please take a look at my [ Ansible Role Development Guidelines](https://github.com/JonasPammer/cookiecutter-ansible-role/blob/master/ROLE_DEVELOPMENT_GUIDELINES.adoc).

If interested, I’ve also written down some [ General Ansible Role Development (Best) Practices](https://github.com/JonasPammer/cookiecutter-ansible-role/blob/master/ROLE_DEVELOPMENT_TIPS.adoc).

## 🔢 Versioning

Versions are defined using [Tags](https://git-scm.com/book/en/v2/Git-Basics-Tagging), which in turn are [recognized and used](https://galaxy.ansible.com/docs/contributing/version.html) by Ansible Galaxy.

**Versions must not start with `v`.**

When a new tag is pushed, [ a GitHub CI workflow](https://github.com/JonasPammer/ansible-role-mediawiki/actions/workflows/release-to-galaxy.yml) (![Release CI](https://github.com/JonasPammer/ansible-role-mediawiki/actions/workflows/release-to-galaxy.yml/badge.svg)) takes care of importing the role to my Ansible Galaxy Account.

## 🧪 Testing

Automatic Tests are run on each Contribution using GitHub Workflows.

The Tests primarily resolve around running [Molecule](https://molecule.readthedocs.io/en/latest/) on a [varying set of linux distributions](#tested-distributions) and using [various ansible versions](#tested-ansible-versions).

The molecule test also includes a step which lints all ansible playbooks using [`ansible-lint`](https://github.com/ansible/ansible-lint#readme) to check for best practices and behaviour that could potentially be improved.

To run the tests, simply run `tox` on the command line. You can pass an optional environment variable to define the distribution of the Docker container that will be spun up by molecule:

    $ MOLECULE_DISTRO=ubuntu2204 tox

For a list of possible values fed to `MOLECULE_DISTRO`, take a look at the matrix defined in [.github/workflows/ci.yml](.github/workflows/ci.yml).

### 🐛 Debugging a Molecule Container

1.  Run your molecule tests with the option `MOLECULE_DESTROY=never`, e.g.:

        $ MOLECULE_DESTROY=never MOLECULE_DISTRO=ubuntu1604 tox -e py3-ansible-5
        ...
          TASK [ansible-role-pip : (redacted).] ************************
          failed: [instance-py3-ansible-9] => changed=false
        ...
         ___________________________________ summary ____________________________________
          pre-commit: commands succeeded
        ERROR:   py3-ansible-9: commands failed

2.  Find out the name of the molecule-provisioned docker container:

        $ docker ps
        30e9b8d59cdf   geerlingguy/docker-debian12-ansible:latest   "/lib/systemd/systemd"   8 minutes ago   Up 8 minutes                                                                                                    instance-py3-ansible-9

3.  Get into a bash Shell of the container, and do your debugging:

        $ docker exec -it 30e9b8d59cdf /bin/bash

        root@instance-py3-ansible-2:/#

    If the failure you try to debug is part of your `verify.yml` step and not the actual `converge.yml`, you may want to know that the output of ansible’s modules (`vars`), hosts (`hostvars`) and environment variables have been stored into files on both the provisioner and inside the docker machine under: \* `/var/tmp/vars.yml` (contains host variables under the `hostvars` key) \* `/var/tmp/environment.yml` `grep`, `cat` or transfer these as you wish!

    You may also want to know that the files mentioned in the admonition above are attached to the **GitHub CI Artifacts** of a given Workflow run.  
    This allows one to check the difference between runs and thus help in debugging what caused the bit-rot or failure in general.

    <figure>
    <img src="https://user-images.githubusercontent.com/32995541/178442403-e15264ca-433a-4bc7-95db-cfadb573db3c.png" alt="178442403 e15264ca 433a 4bc7 95db cfadb573db3c" />
    </figure>

4.  After you finished your debugging, exit it and destroy the container:

        root@instance-py3-ansible-2:/# exit

        $ docker stop 30e9b8d59cdf

        $ docker container rm 30e9b8d59cdf
        or
        $ docker container prune

### 🐛 Debugging installed package versions locally

Although a standard feature in tox 3, this [now](https://github.com/tox-dev/tox/pull/2794) only happens when tox recognizes the presence of a CI variable. For example:

    $ CI=true tox

## 🧃 TIP: Containerized Ideal Development Environment

This Project offers a definition for a "1-Click Containerized Development Environment".

This Container even enables one to run docker containers inside of it (Docker-In-Docker, dind), allowing for molecule execution.

To use it:

1.  Ensure you fullfill the [ the System requirements of Visual Studio Code Development Containers](https://code.visualstudio.com/docs/remote/containers#_system-requirements), optionally following the *Installation*-Section of the linked page section.  
    This includes: Installing Docker, Installing Visual Studio Code itself, and Installing the necessary Extension.

2.  Clone the project to your machine

3.  Open the folder of the repo in Visual Studio Code (*File - Open Folder…*).

4.  If you get a prompt at the lower right corner informing you about the presence of the devcontainer definition, you can press the accompanying button to enter it. **Otherwise,** you can also execute the Visual Studio Command `Remote-Containers: Open Folder in Container` yourself (*View - Command Palette* → *type in the mentioned command*).

I recommend using `Remote-Containers: Rebuild Without Cache and Reopen in Container` once here and there as the devcontainer feature does have some problems recognizing changes made to its definition properly some times.

You may need to configure your host system to enable the container to use your SSH/GPG Keys.

The procedure is described [ in the official devcontainer docs under "Sharing Git credentials with your container"](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials).

## 🍪 CookieCutter

This Project shall be kept in sync with [the CookieCutter it was originally templated from](https://github.com/JonasPammer/cookiecutter-ansible-role) using [cruft](https://github.com/cruft/cruft) (if possible) or manual alteration (if needed) to the best extend possible.

> <figure>
> <img src="https://raw.githubusercontent.com/cruft/cruft/master/art/example_update.gif" alt="Official Example Usage of `cruft update`" />
> </figure>

### 🕗 Changelog

When a new tag is pushed, an appropriate GitHub Release will be created by the Repository Maintainer to provide a proper human change log with a title and description.

## ℹ️ General Linting and Styling Conventions

General Linting and Styling Conventions are [**automatically** held up to Standards](https://stackoverflow.blog/2020/07/20/linters-arent-in-your-way-theyre-on-your-side/) by various [`pre-commit`](https://pre-commit.com/) hooks, at least to some extend.

Automatic Execution of pre-commit is done on each Contribution using [`pre-commit.ci`](https://pre-commit.ci/)[\*](#note_pre-commit-ci). Pull Requests even automatically get fixed by the same tool, at least by hooks that automatically alter files.

Not to confuse: Although some pre-commit hooks may be able to warn you about script-analyzed flaws in syntax or even code to some extend (for which reason pre-commit’s hooks are **part of** the test suite), pre-commit itself does not run any real Test Suites. For Information on Testing, see [🧪 Testing](#testing).

Nevertheless, I recommend you to integrate pre-commit into your local development workflow yourself.

This can be done by cd’ing into the directory of your cloned project and running `pre-commit install`. Doing so will make git run pre-commit checks on every commit you make, aborting the commit themselves if a hook alarm’ed.

You can also, for example, execute pre-commit’s hooks at any time by running `pre-commit run --all-files`.

# 💪 Contributing

![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square) [![Open in Visual Studio Code](https://img.shields.io/static/v1?logo=visualstudiocode&label=&message=Open%20in%20Visual%20Studio%20Code&labelColor=2c2c32&color=007acc&logoColor=007acc)](https://open.vscode.dev/JonasPammer/ansible-role-mediawiki)

The following sections are generic in nature and are used to help new contributors. The actual "Development Documentation" of this project is found under [📝 Development](#development).

## 🤝 Preamble

First off, thank you for considering contributing to this Project.

Following these guidelines helps to communicate that you respect the time of the developers managing and developing this open source project. In return, they should reciprocate that respect in addressing your issue, assessing changes, and helping you finalize your pull requests.

## 🍪 CookieCutter

This Project owns many of its files to [the CookieCutter it was originally templated from](https://github.com/JonasPammer/cookiecutter-ansible-role).

Please check if the edit you have in mind is actually applicable to the template and if so make an appropriate change there instead. Your change may also be applicable partly to the template as well as partly to something specific to this project, in which case you would be creating multiple PRs.

## 💬 Conventional Commits

A casual contributor does not have to worry about following [*the spec*](https://github.com/JonasPammer/JonasPammer/blob/master/demystifying/conventional_commits.adoc) [*by definition*](https://www.conventionalcommits.org/en/v1.0.0/), as pull requests are being squash merged into one commit in the project. Only core contributors, i.e. those with rights to push to this project’s branches, must follow it (e.g. to allow for automatic version determination and changelog generation to work).

## 🚀 Getting Started

Contributions are made to this repo via Issues and Pull Requests (PRs). A few general guidelines that cover both:

-   Search for existing Issues and PRs before creating your own.

-   If you’ve never contributed before, see [ the first timer’s guide on Auth0’s blog](https://auth0.com/blog/a-first-timers-guide-to-an-open-source-project/) for resources and tips on how to get started.

### Issues

Issues should be used to report problems, request a new feature, or to discuss potential changes **before** a PR is created. When you [ create a new Issue](https://github.com/JonasPammer/ansible-role-mediawiki/issues/new), a template will be loaded that will guide you through collecting and providing the information we need to investigate.

If you find an Issue that addresses the problem you’re having, please add your own reproduction information to the existing issue **rather than creating a new one**. Adding a [reaction](https://github.blog/2016-03-10-add-reactions-to-pull-requests-issues-and-comments/) can also help be indicating to our maintainers that a particular problem is affecting more than just the reporter.

### Pull Requests

PRs to this Project are always welcome and can be a quick way to get your fix or improvement slated for the next release. [In general](https://blog.ploeh.dk/2015/01/15/10-tips-for-better-pull-requests/), PRs should:

-   Only fix/add the functionality in question **OR** address wide-spread whitespace/style issues, not both.

-   Add unit or integration tests for fixed or changed functionality (if a test suite already exists).

-   **Address a single concern**

-   **Include documentation** in the repo

-   Be accompanied by a complete Pull Request template (loaded automatically when a PR is created).

For changes that address core functionality or would require breaking changes (e.g. a major release), it’s best to open an Issue to discuss your proposal first.

In general, we follow the "fork-and-pull" Git workflow

1.  Fork the repository to your own Github account

2.  Clone the project to your machine

3.  Create a branch locally with a succinct but descriptive name

4.  Commit changes to the branch

5.  Following any formatting and testing guidelines specific to this repo

6.  Push changes to your fork

7.  Open a PR in our repository and follow the PR template so that we can efficiently review the changes.

# 🗒 Changelog

Please refer to the [Release Page of this Repository](https://github.com/JonasPammer/ansible-role-mediawiki/releases) for a human changelog of the corresponding [Tags (Versions) of this Project](https://github.com/JonasPammer/ansible-role-mediawiki/tags).

Note that this Project adheres to Semantic Versioning. Please report any accidental breaking changes of a minor version update.

# ⚖️ License

**[LICENSE](LICENSE)**

    MIT License

    Copyright (c) 2022, Jonas Pammer

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
