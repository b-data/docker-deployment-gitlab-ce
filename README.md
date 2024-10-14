# GitLab Community Edition (CE)

<!-- markdownlint-disable line-length -->
[![minimal-readme compliant](https://img.shields.io/badge/readme%20style-minimal-brightgreen.svg)](https://github.com/RichardLitt/standard-readme/blob/master/example-readmes/minimal-readme.md) [![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active) <a href="https://liberapay.com/benz0li/donate"><img src="https://liberapay.com/assets/widgets/donate.svg" alt="Donate using Liberapay" height="20"></a>
<!-- markdownlint-enable line-length -->

[This project](https://gitlab.com/b-data/docker/deployments/gitlab-ce) serves as
a template to run [gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce) in a
docker container using docker compose.  
ℹ️ For GitLab on a cloud k8s environment, see GitLab's
[Cloud Native Hybrid reference architectures](https://docs.gitlab.com/ee/administration/reference_architectures/#cloud-native-hybrid).

The docker image is a monolithic image of GitLab running all the necessary
services in a single container.

Features:

* GitLab CE with Mattermost Team Edition (TE) and Container Registry enabled.
  * Disabled: LDAP, Reply by email and Gitlab Pages
  * Includes [gitlab-runner](https://hub.docker.com/r/gitlab/gitlab-runner) to
    register shared runners.
* Pre-configured to run at subdomains of your **own domain**:
  * GitLab: gitlab.mydomain.com
  * Mattermost: mattermost.mydomain.com
  * Container Registry: glcr.mydomain.com
* Exposes GitLab shell on port 10022 by default.
* Sends emails through an
  [exim-relay](https://hub.docker.com/r/devture/exim-relay) container by
  default.
* Use of an .env file for variable substitution in the Compose file.

About GitLab:

* Homepage: <https://about.gitlab.com>
* Documentation: <https://docs.gitlab.com/omnibus/docker/>

## Table of Contents

* [Prerequisites](#prerequisites)
* [Install](#install)
* [Usage](#usage)
* [Further reading](#further-reading)
* [Contributing](#contributing)
* [Support](#support)
* [License](#license)

## Prerequisites

The following is required:

* DNS records for all subdomains pointing to this host.
* Allowing connections on port 10022 to access GitLab shell (Git over SSH).

[Hardware requirements](https://docs.gitlab.com/ee/install/requirements.html#hardware-requirements):

* **Storage:** As a _rule of thumb_ you should have at least as much free space
  as all your repositories combined take up
* **CPU:** **4 cores** is the **recommended** minimum number of cores and
  supports up to 500 users
* **Memory:** **4 GB RAM** is the **required** minimum memory size and supports
  up to 500 users

## Install

This project depends on the following
[Docker Deployment](https://gitlab.com/b-data/docker/deployments):

* [Træfik](https://gitlab.com/b-data/docker/deployments/traefik): TLS
  termination, reverse proxy

## Usage

1. Create an external docker network named "vcs":  

       docker network create vcs

1. Make a copy of all `sample.` files:  

       for file in sample.*; do cp "$file" "${file#sample.}"; done;

1. Update at least environment variables `GL_DOMAIN` and
   `GL_CERTRESOLVER_NAME` in '.env':
    * Replace `mydomain.com` with your **own domain** that serves the
      subdomains.
    * Replace `mydomain-com` with a valid certificate resolvers name of Træfik.
1. Optional: Set these environment variables in '.env':
    * `GL_TZ`: A valid [tz database time zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
      (default: `Etc/UTC`)
    * `GITLAB_SHELL_SSH_PORT`: GitLab Shell SSH port (default: `10022`)
    * `GL_INITIAL_ROOT_PASSWORD`: Initial default admin password (default:
      `password`)
    * `GL_INITIAL_SHARED_RUNNERS_REGISTRATION_TOKEN`: Initial shared runners
      registration token (default: set by GitLab)  
      Generate random registration token:  

          LC_ALL=C tr -cd 'A-Za-z0-9' < /dev/urandom | fold -w 20 | head -n 1

    * `GL_SMTP_PASSWORD`: SMTP server password (disabled)
    * `GL_SMTP_ADDRESS`: SMTP server address (default: `gitlab-smtp`)
    * `GL_SMTP_PORT`: SMTP server port (default: `8025`)
    * `MM_PUBLIC_LINK_SALT`: Mattermost Public Link Salt (default: set by
      GitLab)  
      Generate random salt:  

          LC_ALL=C tr -cd 'a-z0-9' < /dev/urandom | fold -w 32 | head -n 1

1. Optional: Set these environment variables in 'docker-compose.yml':
    * Uncomment line 65 if you have set
      `GL_INITIAL_SHARED_RUNNERS_REGISTRATION_TOKEN` in step 4.
    * Uncomment line 119 if you have set `MM_FILESETTINGS_PUBLICLINKSALT` in
      step 4.
1. Start the container in detached mode:  

       docker compose up -d

### GitLab

Open <https://gitlab.mydomain.com>, log in as user `root` and check the
following settings:

* Admin Area > Settings > General > Visibility and access controls:
  * Default project visibility
  * Default snippet visibility
  * Default group visibility
  * Restricted visibility levels
* Admin Area > Settings > General > Sign-up restrictions:
  * Sign-up enabled
* Admin Area > Settings > General > Sign-in restrictions > Email notification
  for unknown sign-ins:
  * Notify users by email when sign-in location is not recognized
* Admin Area > Settings > Preferences > Localization:
  * Default first day of the week

Change the following settings:

* Admin Area > Settings > General > Third party offers:
  * Tick "Do not display offers from third parties within GitLab"
* Admin Area > Settings > Metrics and profiling > Usage statistics:
  * Untick "Enable usage ping"
* Admin Area > Settings > Network > Outbound requests:
  * Tick "Allow requests to the local network from web hooks and services"
* Admin Area > Settings > Preferences > Email:
  * Untick "Enable in-product marketing emails"

Add Mattermost to Applications:

* Admin Area > Applications: Click "New application"
  * Name: GitLab Mattermost
  * Redirect URL:

        https://mattermost.mydomain.com/signup/gitlab/complete
        https://mattermost.mydomain.com/login/gitlab/complete

    → Replace `mydomain.com` with your own domain that serves the subdomains.
    * Tick "Trusted"
    * Scopes:
      * Tick "api"
* Click "Submit" and copy "Application ID" and "Secret"

### Mattermost

1. Set the following environment variables in '.env':
    * `MM_GITLAB_APPLICATION_ID`: "Application ID" from GitLab
    * `MM_GITLAB_SECRET`: "Secret" from GitLab
1. Reconfigure GitLab:  

       docker compose up -d

1. Wait until GitLab container is ready again.
1. Log into <https://mattermost.mydomain.com> using "GitLab Single Sign-On".

### Register shared runners

    docker exec -ti vcs-gitlab-runner-1 bash -c "gitlab-runner register"

1. Enter your GitLab instance URL:

       Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
       https://gitlab.mydomain.com

1. Enter the token you obtained to register the Runner:

       Please enter the gitlab-ci token for this runner
       <registration token>

1. Enter a description for the Runner, you can change this later in GitLab’s UI:

       Please enter the gitlab-ci description for this runner
       Shared Runner

1. Enter the tags associated with the Runner, you can change this later in
   GitLab’s UI:

       Please enter the gitlab-ci tags for this runner (comma separated):
       <Enter>

1. Enter the Runner executor:
   <!-- markdownlint-disable line-length -->

       Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
       docker

   <!-- markdownlint-enable line-length -->
1. If you chose Docker as your executor, you’ll be asked for the default image
   to be used for projects that do not define one in `.gitlab-ci.yml`:

       Please enter the Docker image (eg. ruby:2.1):
       alpine:latest

See also
[Configuring GitLab Runner](https://docs.gitlab.com/runner/configuration/).

## Further reading

GitLab:

* [Omnibus GitLab Docs](https://docs.gitlab.com/omnibus/)
  * [Setting up LDAP sign-in](https://docs.gitlab.com/ee/administration/auth/ldap/index.html)
  * [SMTP settings](https://docs.gitlab.com/omnibus/settings/smtp.html)  
    → As long as you are using the exim-relay, emails will likely end up in your
    spam folder!
  * [Reply by email](https://docs.gitlab.com/ee/administration/reply_by_email.html)
  * [GitLab Pages administration](https://docs.gitlab.com/ee/administration/pages/)
* [GitLab Runner Docs](https://docs.gitlab.com/runner/)
  * [Advanced configuration](https://docs.gitlab.com/runner/configuration/advanced-configuration.html)

Mattermost:

* [Mattermost Overview](https://docs.mattermost.com/overview/index.html)
  * [Administration Guide](https://docs.mattermost.com/guides/administration.html)
  * [User Guide](https://docs.mattermost.com/guides/channels.html)

## Contributing

PRs accepted. Please submit to the
[GitLab repository](https://gitlab.com/b-data/docker/deployments/gitlab-ce).

This project follows the
[Contributor Covenant](https://www.contributor-covenant.org)
[Code of Conduct](CODE_OF_CONDUCT.md).

## Support

Community support: Open a new discussion
[here](https://github.com/orgs/b-data/discussions).

Commercial support: Contact b-data by [email](mailto:support@b-data.ch).

## License

Copyright © 2019 b-data GmbH

Distributed under the terms of the [MIT License](LICENSE).
