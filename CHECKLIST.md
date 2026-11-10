# Project Submission Checklist – WordPress

This checklist is based on the submission requirements of the Developer Akademie (DevSecOps course). All items have been fulfilled and tested before submission. Additional extras are listed at the end.

## Table of Contents

- [1. Repository](#1-repository)
  - [Existing Files](#existing-files)
  - [docker-compose.yaml](#docker-composeyaml)
  - [README.md](#readmemd)
- [2. Documentation](#2-documentation)
- [3. Notes](#3-notes)
  - [Security](#security)
  - [Code Conventions](#code-conventions)
  - [Testing](#testing)
- [Extras](#extras)

## 1. Repository

### Existing Files

- [x] A `.gitignore` file exists that ignores all irrelevant content of the git repository
- [x] A `docker-compose.yaml` exists that meets the requirements of the next section
- [x] A file named `README.md` exists and was written according to the criteria below
- [x] There are no other files in the repository unless they are explicitly named and described in the `README.md` (`.env.example`, `CHECKLIST.md`)

### docker-compose.yaml

- [x] Two services are defined: `wordpress` and `db`
- [x] There is an environment configuration for the `wordpress` service in which all non-critical variables are configured (no auth!)
- [x] There are volume configurations for the database so that its content is persisted on a file system (`db_data`)
- [x] Both services run in the same network (`wordpress_net`)

### README.md

- [x] The README contains a table of contents (ToC)
- [x] There is a section describing the repository: its main contents and its purpose
- [x] There is a "Quickstart" section with the prerequisites and a short quickstart guide
- [x] There is a detailed version of the quickstart as "Usage", explaining the configuration and how relevant parts can be modified to achieve different results

## 2. Documentation

- [x] The code and the project are documented in the repository in the form of a README file
- [x] The documentation is written in English

## 3. Notes

### Security

- [x] No SSH keys are stored in the workspace of the git repository
- [x] No passwords, tokens, or usernames are stored in the code; environment variables are used instead
- [x] No IP addresses or other sensitive information are stored in the git repository

### Code Conventions

- [x] Build args, environment variables, and shell variables follow the naming convention `UPPER_CASE_WITH_UNDERSCORE`
- [x] Variables are always referenced with the `{}` notation: `${SOME_VAR_VALUE}` instead of `$SOME_VAR_VALUE`
- [x] Default values are configured for environment variables where it makes sense (`${VAR:-default}`)
- [x] Critical configuration such as passwords is not stored in the repository but passed into the containers via a `.env` file

### Testing

Tested on the cloud VM before submission:

- [x] WordPress is reachable at the IP address of the cloud VM on port `8080`
- [x] It is possible to log in with the configured admin user credentials and navigate WordPress
- [x] After restarting the setup, the configured data is still present and is not deleted or overwritten
- [x] The service containers are restarted as soon as an error occurs that terminates the container (verified via `RestartCount` after killing the container process)

## Extras

- Database healthcheck combined with `depends_on: condition: service_healthy` for a reliable startup order
- Additional volume `wp_data`, so uploads, themes, plugins, and `wp-config.php` persist as well
- Database port is not exposed to the host, only reachable within the internal network
- Image versions and the published port are configurable via environment variables with defaults
