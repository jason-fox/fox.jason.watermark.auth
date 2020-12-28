# Watermark Auth Plugin for DITA-OT

[![license](https://img.shields.io/github/license/jason-fox/fox.jason.watermark.auth.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![DITA-OT 3.6](https://img.shields.io/badge/DITA--OT-3.6-blue.svg)](http://www.dita-ot.org/3.6)
[![Documentation Status](https://readthedocs.org/projects/watermarkdita-ot/badge/?version=latest)](https://watermarkdita-ot.readthedocs.io/en/latest/?badge=latest)

This is an example [DITA-OT Plug-in](https://www.dita-ot.org/plugins) to authorize the watermarking of generated PDF
files. The plugin uses the extension-point offered by the
[DITA-OT Watermark plug-in](https://github.com/jason-fox/fox.jason.watermark) to make a request to an authorization
server to decide whether a user has rights to create a document of the given type.

> **Note** The example `authorize-user` macro in [`resource/antlib.xml`](./resource/antlib.xml) is not connecting to a
> valid authorization server and therefore the code will need to be updated to work with a live system.

<details>
<summary><strong>Table of Contents</strong></summary>

-   [Install](#install)
    -   [Installing DITA-OT](#installing-dita-ot)
    -   [Installing the Plug-in](#installing-the-plug-in)
      -   [Connecting to Authorization service](#connecting-to-authorization-service)
    -   [Building the Authorization service as a Docker container](#building-the-authorization-service-as-a-docker-container)
        -   [Using the Dummy IDM server](#using-the-dummy-idm-server)
        -   [Using a real IDM service (Keyrock)](#using-a-real-idm-service-keyrock)
-   [Usage](#usage)
    -   [Parameter Reference](#parameter-reference)
-   [License](#license)

</details>

## Install

The DITA-OT Water plug-in has been tested against [DITA-OT 3.x](http://www.dita-ot.org/download). It is recommended that
you upgrade to the latest version.

### Installing DITA-OT

<a href="https://www.dita-ot.org"><img src="https://www.dita-ot.org/images/dita-ot-logo.svg" align="right" height="55"></a>

The DITA-OT Watermark Authorization plug-in is a plug-in for the DITA Open Toolkit.

-   Full installation instructions for downloading DITA-OT can be found
    [here](https://www.dita-ot.org/3.6/topics/installing-client.html).

    1.  Download the `dita-ot-3.6.zip` package from the project website at
        [dita-ot.org/download](https://www.dita-ot.org/download)
    2.  Extract the contents of the package to the directory where you want to install DITA-OT.
    3.  **Optional**: Add the absolute path for the `bin` directory to the _PATH_ system variable.

    This defines the necessary environment variable to run the `dita` command from the command line.

```console
curl -LO https://github.com/dita-ot/dita-ot/releases/download/3.6/dita-ot-3.6.zip
unzip -q dita-ot-3.6.zip
rm dita-ot-3.6.zip
```

### Installing the Plug-in

-   Run the plug-in installation commands:

```console
dita install https://github.com/jason-fox/fox.jason.watermark/archive/master.zip
dita install https://github.com/jason-fox/fox.jason.watermark.auth/archive/master.zip
```

#### Connecting to Authorization service

Two settings are placed in the `cfg/configuration.properties` file. These Currently this just send a data to a simple
authorization service defined below, which is assumed to be running on `localhost:3000` - amend the settings as
necessary when hosting the service elsewhere.

```text
auth.token.url=http://localhost:3000/login
auth.pep.url=http://localhost:3000
```

### Building the Authorization service as a Docker container

```console
cd server
docker build -t authservice .
```

#### Using the Dummy IDM server

An simple authorization service has been added to the repository. By default this connects to a dummy Identity Manager
server (IDM) . To start the authorization server run:

```console
docker run -d --rm  -p 3000:3000 --env DEBUG='server:*' authservice
```

The Dummy IDM recognises the following users passwords and tokens:

-   `alice` - password: `auth`. token: `eec3954b328ad6f73d33efcb93a96e67`.
-   `bob` - password: `bobspassword`. token: `627fa25c7972302db34a7caaa35aa33b`.
-   `charlie` - password: `test`. token: `7993341a1f9f850ee8988bac2df3af7d`.
-   `eve` - password: `secret`. token: `2c12606b1d356f4e51429957a4defc4f`.

Alice and Bob have full authorization rights, Charlie `draft` and `review` and Eve can `draft` only.

#### Using a real IDM service (Keyrock)

[Keyrock](https://fiware-idm.readthedocs.io/) a full Identity Management server which offers OAuth2-based authentication
and authorization interfaces.

More information on Keyrock can be found [here](https://github.com/FIWARE/tutorials.Identity-Management). Once a
`IDM_CLIENT_ID` and `IDM_CLIENT_SECRET` have been created user management can be delegated to Keyrock by altering the
`config.js` or by adding the following `ENV`s on start-up

```console
docker run -d --rm  -p 3000:3000 --env DEBUG='server:*' \
  --env IDM_PORT=<port> \
  --env IDM_URL=<url> \
  --env IDM_IP_ADDRESS=<url>  \
  --env IDM_CLIENT_ID=<client-id> \
  --env IDM_CLIENT_SECRET=<client-secret> \
 authservice
```

Since Keyrock follows the OAuth2 standard for its responses, it should be simple to amend the existing code to point to
any other OAuth2 service if necessary.

By default actions are defined as `GET` resources of the form `<product>/<action>` which require authorization. Every
`<action>` may also be whitelisted as an authenticate action (which only require a valid token) or a permitted action
(which is not checked) by adding the `AUTHENTICATE_ACTIONS` and `PERMIT_ACTIONS` environment variables holding a comma
delimited list of values.

---

## Usage

The plugin extends standard PDF processing:

```console
PATH_TO_DITA_OT/bin/dita -f pdf -o out -i document.ditamap --auth.level=draft|review|final --auth.token=<token>
```

`auth.token` is an OAuth2 access token. if the provided access token is valid, the preferred watermark will be used
assuming the user has the rights to apply it. As a fallback, the output PDF will be watermarked as a **DRAFT**. If no
`auth.token` is supplied, an `auth.user` and `auth.password` can be supplied instead to request a new OAuth2 access
token on the fly.

### Parameter Reference

-   `auth.level` - Decides which watermark to use:
    -   `draft` - Adds a watermark stating **DRAFT DOCUMENT** (default)
    -   `review` - Adds a watermark stating _Copy for review only_
    -   `final` - Adds an invisible watermark
    -   `none` - Does not add a watermark
-   `auth.password` - Password for Identity Management server
-   `auth.user` - Username for Identity Management server
-   `auth.token` - OAuth2 access token for authorization

## License

[Apache 2.0](LICENSE) © 2020 Jason Fox
