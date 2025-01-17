---
order: 0
summary: How to configure the Denoflare CLI
title: Configuration
---

# .denoflare configuration file
How to configure the Denoflare CLI

Most commands can be run using cli options, but sometimes it's easier to specify profiles and script defs in a .denoflare configuration file.

Specify your configuration in a jsonc file, and either
 - Specify it explicitly: e.g. `denoflare --config /path/to/my-config.jsonc`
 - Or save it with the filename `.denoflare` either in the current working directory, or any ancestors (e.g. `~/.denoflare` is common) and it will be used implicitly

## Example .denoflare file
```jsonc
{
	// This jsonc file supports comments and trailing commas!

    // optional schema to get auto-completions when editing this file in vscode, etc
	"$schema": "https://raw.githubusercontent.com/skymethod/denoflare/v0.3.3/common/config.schema.json",

    // define script configurations by name, and their associated bindings and options
    // you can then simply refer to them by name in your denoflare commands
    // note the same script source can be specified more than once with different configurations (local vs prod etc)
	"scripts": {
		"script1-local": {
			"path": "/Users/me/path/to/script1.js",
			"bindings": {
				"foo": { "value": "bar" },
			},
			"localPort": 3002,
		},
		"script2-local": {
			"path": "/Users/me/path/to/script2.ts",
			"bindings": {
				"memoryNamespace": { "doNamespace": "local:MyDO" }
			},
			"localPort": 8080,
		},
		"script3-local": {
			"path": "/Users/me/path/to/script3.ts",
			"bindings": {
				"MyKV": { "kvNamespace": "351bddca004649b5a58c985b0gus86d5" },
				"LocalHostAndPort": { "value": "localhost:${localPort}" }
			},
			"localPort": 3031,
			"localHostname": "test.example.com",
			"localIsolation": "isolate"
		},
		"script4-prod": {
		    "path": "/Users/me/path/to/script4.ts",
		    "bindings": {
				"version": { "value": "1.2.3" },
			},
		},
		"script4-local": {
		    "path": "/Users/me/path/to/script4.ts", // same script, different binding value
		    "bindings": {
				"version": { "value": "local" },
			},
		},
	},

    // define one or more named profiles to specify Cloudflare API token credentials
    // you can then simply refer to them by name in your denoflare commands, using --profile <profile-name>
    // or if no --profile is specified, the default profile will be used
    // the default profile is either the only profile specified (as shown here), or the one marked with default: true
	"profiles": {
		"account1": {
			"accountId": "4fc7a4becb8b41d887c5bb970b0guse1",
			"apiToken": "kAE-bogUs-_d8fkg2kkjc8dsftq63jda9kgs78"
		}
	}
}
```

## Full configuration format
```ts
/** Top-level Denoflare configuration object, typically saved in a `.denoflare` file. */
export interface Config {

    /** Known script definitions, by unique `script-name`.
     * 
     * `script-name` must:
     *  - start with a letter
     *  - end with a letter or digit
     *  - include only lowercase letters, digits, underscore, and hyphen
     *  - be 63 characters or less
     * 
     * Only the name must be unique, multiple Script definitions can point to the same worker `path` but different Bindings.
     * This is useful when defining multiple environments for the same script, named `worker-local`, `worker-dev`, `worker-prod`, etc.
     */
    readonly scripts?: Record<string, Script>;

    /** Profile definitions by unique `profile-name`.
     * 
     * `profile-name` must:
     *  - start with a letter
     *  - end with a letter or digit
     *  - include only lowercase letters, digits, and hyphen
     *  - be 36 characters or less
     */
    readonly profiles?: Record<string, Profile>;
}

/** Code isolation to use when running worker scripts with `serve`, the local dev server.
 * 
 * - `'none'`:    Run the worker script in the same isolate as the host.
 *                Easier to debug, but no hot reloads, and same Deno permissions as the host.
 * - `'isolate'`: (default) Run the worker script in a separate isolate (webworker) with no Deno permissions.
 *                Harder to debug, but can be hot reloaded on script changes, and safer.
 */
export type Isolation = 'none' | 'isolate';

/** Script-level configuration */
export interface Script {

    /** (required) Local file path, or https: url to a module-based worker entry point .ts, or a non-module-based worker bundled .js */
    readonly path: string;

    /** Bindings for worker environment variables to use when running locally, or deploying to Cloudflare */
    readonly bindings?: Record<string, Binding>;

    /** If specified, use this port when running `serve`, the local dev server. */
    readonly localPort?: number;

    /** If specified, replace the hostname portion of the incoming `Request.url` at runtime to use this hostname instead of `localhost`.
     * 
     * Useful if your worker does hostname-based routing. */
    readonly localHostname?: string;

    /** If specified, use this isolation level when running `serve`, the local dev server.
     * 
     * (Default: 'isolate') */
    readonly localIsolation?: Isolation;

    /** If specified, use a specific, named Profile defined in `config.profiles`.
     * 
     * (Default: the Profile marked as `default`, or the only Profile defined) */
    readonly profile?: string;
}

/** Binding definition for a worker script environment variable */
export type Binding = TextBinding | SecretBinding | KVNamespaceBinding | DONamespaceBinding;

/** Plain-text environment variable binding */
export interface TextBinding {

    /** Value is the string value, with the following replacements:
     *  - `${localPort}` replaced with the localhost port used when running `serve`, the local dev server.
     *    This can be useful when defining a variable for the server Origin, for example.
     */
    readonly value: string;
}

/** Secret-text environment variable binding */
export interface SecretBinding {

    /** Value can be:
     *  - Secret literal string value
     *  - `aws:<aws-profile-name>`, replaced with the associated `<aws_access_key_id>:<aws_secret_access_key>` from `~/.aws/credentials`.
     *    Useful if you want to keep your credentials in a single file.
     */
    readonly secret: string;
}

/** Workers KV Namespace environment variable binding */
export interface KVNamespaceBinding {

    /** For now, this is the underlying Cloudflare API ID of the Workers KV Namespace. */
    readonly kvNamespace: string;
}

/** Workers Durable Object Namespace environment variable binding */
export interface DONamespaceBinding {

    /** For now, this is either:
     * - The underlying Cloudflare API ID of the Workers Durable Object Namespace
     * - `local:<DOClassName>`: Pointer to a Durable Object class name defined in the same worker script. e.g. `local:MyCounterDO`
     */
    readonly doNamespace: string;
}

/** Profile definition, Cloudflare credentials to use when deploying via `push`, or running locally with `serve` using real KV storage. */
export interface Profile {

    /** Cloudflare Account ID: 32-char hex string.
     * 
     * This value can either be specified directly, or using `regex:<file-path>:<pattern-with-capturing-group>` to grab the value from another file.
     */
    readonly accountId: string;

    /** Cloudflare API token: Value obtained from the Cloudflare dashboard (My Profile -> [API Tokens](https://dash.cloudflare.com/profile/api-tokens)) when creating the token under this account. 
     * 
     * This value can either be specified directly, or using `regex:<file-path>:<pattern-with-capturing-group>` to grab the value from another file.
     */
    readonly apiToken: string;

    /** If there are multiple profiles defined, choose this one as the default (when no `--profile` is explicitly specified or configured). 
     * 
     * There can only be one default profile.
    */
    readonly default?: boolean;
}
```
