# Quickstart

## Prerequisites

- A basic understanding of [HashiCorp Vault](https://www.hashicorp.com/products/vault) (see [What is Vault?](https://developer.hashicorp.com/vault/docs/what-is-vault) for details).
- A [HashiCorp Vault server](https://developer.hashicorp.com/vault/docs/install).
  - **Note:** This guide also includes [quick start instructions](#quick-start-for-evaluation) for running a Vault server in development mode if you'd like to evaluate this plugin before starting to use it.
- The [HashiCorp Vault CLI](https://developer.hashicorp.com/vault/downloads) installed on your device.
- A [1Password Connect server](https://developer.1password.com/docs/connect/get-started/) deployed on your infrastructure.
- [Go](https://go.dev/doc/install) (if you want to build the plugin from source).

## Quick start (for evaluation)

You can start HashiCorp Vault server in [development mode](https://developer.hashicorp.com/vault/docs/concepts/dev-server) to demonstrate and evaluate the 1Password secrets engine. Vault starts unsealed in this configuration, and you do not need to register the plugin.

```sh
vault server -dev -dev-root-token-id=root -dev-plugin-dir=./vault/plugins -log-level=debug
```

> **Warning:** Running Vault in development mode is useful for evaluating the plugin, but should **never** be used in production.

Connect to the Vault server in a **new** terminal to [enable the secrets engine](#enable-and-configure-the-plugin) and start using it.

## Getting started

### Download the binary

1. Download [the release for your HashiCorp Vault server's architecture](https://github.com/1Password/vault-plugin-secrets-onepassword/releases).
1. Extract the binary and move it to the [plugin directory](https://developer.hashicorp.com/vault/docs/plugins/plugin-architecture#plugin-directory) for your Vault server.

Example for a Vault server running on a Linux machine with AMD64 architecture (note that you may need to update the filename depending on which OS and architecture you're using):

```sh
# Unzip the release archive
unzip ./vault-plugin-secrets-onepassword_1.1.0_linux_amd64.zip
# Create a plugin directory
mkdir -p ./vault/plugins
# Move the binary to the plugin directory and rename to op-connect
mv ./vault-plugin-secrets-onepassword_v1.1.0 ./vault/plugins/op-connect
```

### Build the binary

Alternatively, you can also build the plugin binary from source:

```sh
# Clone this repository
git clone https://github.com/1Password/vault-plugin-secrets-onepassword.git
# Build the binary
go build -o ../vault/plugins/op-connect -C ./vault-plugin-secrets-onepassword ./main.go
```

## Full installation

### Configure and start the Vault server

Write the [configuration file](https://developer.hashicorp.com/vault/docs/configuration) for the Vault server. For example:

```sh
cat > ./vault/server.hcl << EOF
plugin_directory = "$(pwd)/vault/plugins"
api_addr         = "http://127.0.0.1:8200"

storage "inmem" {}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = "true"
}
EOF
```

> **Note:** You must set [`plugin_directory`](https://developer.hashicorp.com/vault/docs/configuration#plugin_directory) to point to the folder with your custom secrets engine and [`api_addr`](https://developer.hashicorp.com/vault/docs/configuration#api_addr) for the plugin to communicate with Vault. The Vault instance stores everything in memory and runs locally.

Start the Vault server with this configuration file:

```sh
vault server -config=./vault/server.hcl
```

Finally, connect to the Vault server in a **new** terminal.

### Register the plugin to Vault

Initialize and unseal Vault:

```sh
# Set VAULT_ADDR to connect to the local Vault server
export VAULT_ADDR='http://127.0.0.1:8200'
# Initialize Vault
vault operator init
# Unseal Vault with the unseal key
vault operator unseal
```

Next, calculate the SHA256 checksum of the plugin binary. For example, on Linux:

```sh
SHA256_CHECKSUM=$(sha256sum ./vault/plugins/op-connect | cut -d ' ' -f1)
```

Or, on macOS:

```sh
SHA256_CHECKSUM=$(shasum -a 256 .vault/plugins/op-connect | cut -d ' ' -f1)
```

Next, register the plugin to the catalog for the Vault server:

```sh
vault plugin register -sha256=$SHA256_CHECKSUM secret op-connect
```

### Enable and configure the plugin

Enable the `op-connect` secrets engine at the `op/` path:

```sh
vault secrets enable --path="op" op-connect
```

> **Note:** You will need to provide the URL for your 1Password Connect server and your Connect access token for the next steps.

Write the configuration data to access your Connect server to `op/config` in a single command (assuming the `OP_CONNECT_` environment variables have been set):

```sh
vault write op/config \
  op_connect_host=$OP_CONNECT_HOST \
  op_connect_token=$OP_CONNECT_TOKEN
```

Alternatively, create a JSON file with your 1Password Connect server details. For example, save the following as `op-connect-config.json`:

```json
{
	"op_connect_host": "https://op-connect.example.com:8443/",
	"op_connect_token": "your_access_token"
}
```

Write the data to the `op/config` path using this file to configure the secrets engine to access 1Password Connect server.

```sh
vault write op/config @op-connect-config.json
```

## Usage

### Environment variables

- `OP_CONNECT_TOKEN`: (required if `op_connect_token` is not set in configuration): The API token used to authenticate with the 1Password Connect API.

### Commands

> **Note:** When specifying the 1Password vault name or item title in the path, if multiple vaults or items have the same respective name or title, the action will be performed on the _oldest_ vault or item. Item titles or vault names that include white space characters **cannot** be used (you can reference them using UUID instead).

#### List vaults

Returns the names and UUIDs for the vault(s) that are accessible to the Connect access token:

```sh
vault list op/vaults
```

#### List items

Returns the title and UUID for items stored in the specified 1Password vault:

```sh
vault list op/vaults/<vault_name_or_uuid>
```

#### Read item

Returns the data for the specified item:

```sh
vault read op/vaults/<vault_name_or_uuid>/items/<item_title_or_uuid>
```

#### Create item

Create an item from a JSON file (see [Details for creating and updating items](#details-for-creating-and-updating-items) for more information on the JSON schema):

```sh
vault write op/vaults/<vault_name_or_uuid>/items @some_json_file.json
```

#### Update item

Update the specified item using a JSON file (see [Details for creating and updating items](#details-for-creating-and-updating-items) for more information on the JSON schema):

```sh
vault write op/vaults/<vault_name_or_uuid>/items/<item_title_or_uuid> @some_json_file.json
```

#### Delete item

Delete the specified item:

```sh
vault delete op/vaults/<vault_name_or_uuid>/items/<item_title_or_uuid>
```

#### Details for creating and updating items

See [1Password Connect Server API Reference](https://developer.1password.com/docs/connect/connect-api-reference) for more details:

- `category` (**required**): the category of the item to create (`database`, `login`, and `password` are currently supported)
- `title` (**required** on create): a name for the item
- `url`: the URL where the item may be filled
- `fields`: an array of fields to create for the item; each field can be described with the following:
  - `id`: the ID of the field to create
  - `label`: the field name displayed in 1Password apps
  - `type`: the type of the field (`STRING`, `EMAIL`, `CONCEALED`, `URL`, `TOTP`, `DATE`, `MONTH_YEAR`, and `MENU` are currently supported)
  - `purpose`: the purpose of the field (`""`, `USERNAME`, `PASSWORD`, `NOTES` are currently supported)
  - `value`: the value stored in the field
  - `generate` (used for `PASSWORD` fields): set to `true` to have 1Password generate the value.
  - `entropy` (used for `PASSWORD` fields): set to an integer entropy value
  - `section`: the section to place the field; if not specified, the field will be placed in the default section
    - `id`: the ID of the section
- `sections`: an array of sections to create for the item; each section can be described with the following:
  - `id`: an ID for the section
  - `label`: how the section will be titled in the UI

#### Example Login item with custom section

```json
{
	"category": "login",
	"title": "Example Login",
	"fields": [
		{
			"id": "username",
			"label": "username",
			"type": "STRING",
			"purpose": "USERNAME",
			"value": "my_user"
		},
		{
			"id": "password",
			"label": "password",
			"purpose": "PASSWORD",
			"type": "CONCEALED",
			"value": "",
			"generate": true
		},
		{
			"id": "custom_field_id",
			"type": "STRING",
			"label": "My Custom Field",
			"value": "my custom value",
			"section": {
				"id": "my_new_section"
			}
		}
	],
	"sections": [
		{
			"id": "my_new_section",
			"label": "New Section"
		}
	]
}
```

#### Example Password item

```json
{
	"category": "password",
	"title": "Example Password",
	"fields": [
		{
			"id": "password",
			"label": "password",
			"purpose": "PASSWORD",
			"type": "CONCEALED",
			"value": "",
			"generate": true
		}
	]
}
```

#### Example Database item

```json
{
	"category": "database",
	"title": "Example Database",
	"fields": [
		{
			"id": "username",
			"label": "username",
			"type": "STRING",
			"purpose": "USERNAME",
			"value": "my_user"
		},
		{
			"id": "password",
			"label": "password",
			"purpose": "PASSWORD",
			"type": "CONCEALED",
			"value": "",
			"generate": true
		},
		{
			"id": "hostname",
			"label": "hostname",
			"type": "STRING",
			"value": "my_host"
		},
		{
			"id": "database",
			"label": "database",
			"type": "STRING",
			"value": "my_database"
		},
		{
			"id": "port",
			"label": "port",
			"type": "STRING",
			"value": "8080"
		}
	]
}
```

## Vault Enterprise Namespaces

The 1Password secrets engine supports [Vault Enterprise Namespaces](https://developer.hashicorp.com/vault/docs/enterprise/namespaces). If you are using namespaces, the secrets engine must be enabled for each namespace.

You can use `-namespace` to enable the plugin for each namespace. For example:

```sh
vault secrets enable -namespace=ns_example op
```

The plugin also requires configuration in the namespace. Write the configuration in a single command:

```sh
vault write -namespace=ns_example op/config \
  op_connect_host=$OP_CONNECT_HOST \
  op_connect_token=$OP_CONNECT_TOKEN
```

Alternatively, you can also create a file for the configuration and write the contents to the path (see [Enable and configure the plugin](#enable-and-configure-the-plugin) above for an example file):

```sh
vault write -namespace=ns_example op/config @op-connect-config.json
```
