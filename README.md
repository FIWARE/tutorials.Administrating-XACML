[![FIWARE Banner](https://fiware.github.io/tutorials.Adminstrating-XACML/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://www.fiware.org/developers/catalogue/)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Adminstrating-XACML.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

This tutorial introduces the adminstration of level 3 advanced authorization rules into **Keyrock**. The simple verb-resource based permissions are amended to use XACML and new XACML permissions added to the existing roles. The updated ruleset is automatically uploaded to **Authzforce** PDP, so that policy execution points such as the **PEP proxy** are able to apply the latest ruleset.

The tutorial demonstrates examples of interactions using the **Keyrock** GUI, as
well [cUrl](https://ec.haxx.se/) commands used to access the REST
APIs of **Keyrock**  and **Authzforce** -
[Postman documentation](https://fiware.github.io/tutorials.Identity-Management/)
is also available.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://www.getpostman.com/collections/66d8ba3abaf7319941b1)

# Contents

- [Administrating XACML Rules](#administrating-xacml-rules)
  * [What is XACML](#what-is-xacml)
- [Prerequisites](#prerequisites)
  * [Docker](#docker)
  * [Cygwin](#cygwin)
- [Architecture](#architecture)
- [Start Up](#start-up)
    + [Dramatis Personae](#dramatis-personae)
- [XACML Administration](#xacml-administration)
  * [Authzforce - Administrating XACML PolicySets](#authzforce---administrating-xacml-policysets)
    + [Creating a new Domain](#creating-a-new-domain)
    + [Creating an initial PolicySet](#creating-an-initial-policyset)
    + [Activating the initial PolicySet](#activating-the-initial-policyset)
    + [Updating a PolicySet](#updating-a-policyset)
    + [Activating an updated PolicySet](#activating-an-updated-policyset)
  * [Keyrock - Administrating XACML Permissions](#keyrock---administrating-xacml-permissions)
    + [Create Token with Password](#create-token-with-password)
    + [Update an XACML Permission](#update-an-xacml-permission)
- [PEP Proxy - Extending Advanced Authorization](#pep-proxy---extending-advanced-authorization)
  * [PEP Proxy - Sample Code](#pep-proxy---sample-code)
  * [PEP Proxy - Running the Example](#pep-proxy---running-the-example)

# Administrating XACML Rules

> **12.3 Central Terminal Area**
>
> * Red or Yellow Zone
>    * No private vehicle shall stop, wait, or park in the red or yellow zone.
> * White Zone
>    * No vehicle shall stop, wait, or park in the white zone unless actively
> engaged in the immediate loading or unloading of passengers
> and/or baggage. 
>
> — Los Angeles International Airport Rules and Regulations, Section 12 - Landside Motor Vehicle Operations



TBD

## What is XACML



# Prerequisites

## Docker

To keep things simple all components will be run using
[Docker](https://www.docker.com). **Docker** is a container technology which
allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions
    [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions
    [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions
    [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker
applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml)
is used configure the required services for the application. This means all
container services can be brought up in a single command. Docker Compose is
installed by default as part of Docker for Windows and Docker for Mac, however
Linux users will need to follow the instructions found
[here](https://docs.docker.com/compose/install/)

## Cygwin

We will start up our services using a simple bash script. Windows users should
download [cygwin](http://www.cygwin.com/) to provide a command-line
functionality similar to a Linux distribution on Windows.

# Architecture

This application adds OAuth2-driven security into the existing Stock Management
and Sensors-based application created in
[previous tutorials](https://github.com/Fiware/tutorials.IoT-Agent/) by using
the data created in the first
[security tutorial](https://github.com/Fiware/tutorials.Identity-Management/)
and reading it programmatically. It will make use of three FIWARE components -
the [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/),the
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
and integrates the use of the
[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic enabler. Usage
of the Orion Context Broker is sufficient for an application to qualify as
_“Powered by FIWARE”_.

Both the Orion Context Broker and the IoT Agent rely on open source
[MongoDB](https://www.mongodb.com/) technology to keep persistence of the
information they hold. We will also be using the dummy IoT devices created in
the [previous tutorial](https://github.com/Fiware/tutorials.IoT-Sensors/).
**Keyrock** uses its own [MySQL](https://www.mysql.com/) database.

Therefore the overall architecture will consist of the following elements:

-   The FIWARE
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which
    will receive requests using
    [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
-   The FIWARE
    [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
    which will receive southbound requests using
    [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) and convert
    them to
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    commands for the devices
-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) offer a
    complement Identity Management System including:
    -   An OAuth2 authentication system for Applications and Users
    -   A site graphical frontend for Identity Management Administration
    -   An equivalent REST API for Identity Management via HTTP requests

-   FIWARE [Authzforce](https://fiware-pep-proxy.rtfd.io/) is a XACML Server providing an interpretive Policy Decision Point (PDP)
    access to the **Orion** and/or **IoT Agent** microservices
-   FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a PEP Proxy securing
    access to the **Orion** microservices, it requests authorisation decisions from **Authzforce**
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the **Orion Context Broker** to hold context data information
        such as data entities, subscriptions and registrations
    -   Used by the **IoT Agent** to hold device information such as device URLs
        and Keys
-   A [MySQL](https://www.mysql.com/) database :
    -   Used to persist user identities, applications, roles and permissions
-   The **Stock Management Frontend** does the following:
    -   Displays store information
    -   Shows which products can be bought at each store
    -   Allows users to "buy" products and reduce the stock count.
    -   Allows authorized users into restricted areas, it requests authoriation decisions from **Authzforce**
-   A webserver acting as set of
    [dummy IoT devices](https://github.com/Fiware/tutorials.IoT-Sensors) using
    the
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    protocol running over HTTP - access to certain resources is restricted.

Since all interactions between the elements are initiated by HTTP requests, the
entities can be containerized and run from exposed ports.

![](https://fiware.github.io/tutorials.Adminstrating-XACML/img/architecture.png)


The all container configuration values described in the YAML file
have been described in previous tutorials



# Start Up

To start the installation, do the following:

```console
git clone git@github.com:Fiware/tutorials.Adminstrating-XACML.git
cd tutorials.Adminstrating-XACML

./services create
```

> **Note** The initial creation of Docker images can take up to three minutes

Thereafter, all services can be initialized from the command-line by running the
[services](https://github.com/Fiware/tutorials.Adminstrating-XACML/blob/master/services)
Bash script provided within the repository:

```console
./services start
```


> :information_source: **Note:** If you want to clean up and start over again
> you can do so with the following command:
>
> ```console
> ./services stop
> ```

### Dramatis Personae

The following people at `test.com` legitimately have accounts within the
Application

-   Alice, she will be the Administrator of the **Keyrock** Application
-   Bob, the Regional Manager of the supermarket chain - he has several store
    managers under him:
    -   Manager1
    -   Manager2
-   Charlie, the Head of Security of the supermarket chain - he has several
    store detectives under him:
    -   Detective1
    -   Detective2

| Name       | eMail                     | Password |
| ---------- | ------------------------- | -------- |
| alice      | alice-the-admin@test.com  | `test`   |
| bob        | bob-the-manager@test.com  | `test`   |
| charlie    | charlie-security@test.com | `test`   |
| manager1   | manager1@test.com         | `test`   |
| manager2   | manager2@test.com         | `test`   |
| detective1 | detective1@test.com       | `test`   |
| detective2 | detective2@test.com       | `test`   |

The following people at `example.com` have signed up for accounts, but have no
reason to be granted access

-   Eve - Eve the Eavesdropper
-   Mallory - Mallory the malicious attacker
-   Rob - Rob the Robber

| Name    | eMail               | Password |
| ------- | ------------------- | -------- |
| eve     | eve@example.com     | `test`   |
| mallory | mallory@example.com | `test`   |
| rob     | rob@example.com     | `test`   |


# XACML Administration
## Authzforce - Administrating XACML PolicySets

### Creating a new Domain

### Creating an initial PolicySet

### Activating the initial PolicySet

### Updating a PolicySet

### Activating an updated PolicySet

## Keyrock - Administrating XACML Permissions

### Create Token with Password

### Update an XACML Permission


# PEP Proxy - Extending Advanced Authorization

## PEP Proxy - Sample Code

## PEP Proxy - Running the Example

---

## License

[MIT](LICENSE) © FIWARE Foundation e.V.
