# Common Environment Variables

There are two types of environment variables used by all EdgeX services. They are `standard` and `overrides`. The only difference is that the `overrides` apply to command-line options and service configuration settings where as `standard` do not have any corresponding command-line option or configuration setting.

## Standard Environment Variables

This section describes the `standard` environment variables common to all EdgeX services. Some service may have additional  `standard` environment variables which are documented in those service specific sections.

#### EDGEX_SECURITY_SECRET_STORE

This environment variables indicates whether the service is expected to initialize the secure SecretStore which allows the service to access secrets from Vault. Defaults to `true` if not set or not set to `false`. When set to `true` the EdgeX security services must be running. If running EdgeX in `non-secure` mode you then want this explicitly set to `false`.

!!! example "Example - Using docker-compose to disable secure SecretStore"
    ```yaml
    environment: 
      EDGEX_SECURITY_SECRET_STORE: "false"
    ```

#### EDGEX_DISABLE_JWT_VALIDATION

This environment variable disables, at the microservice-level, validation of the `Authorization` HTTP header of inbound REST API requests.
(Microservice-level authentication was added in EdgeX 3.0.)

Normally, when `EDGEX_SECURITY_SECRET_STORE` is unset or `true`,
EdgeX microservices authenticate inbound HTTP requests by parsing the `Authorization` header,
extracting a JWT bearer token,
and validating it with the EdgeX secret store,
returning an HTTP 401 error if token validation fails.

If for some reason it is not possible to pass a valid JWT to an EdgeX microservice --
for example, the eKuiper rule engine making an unauthenticated HTTP API call, or other legacy code --
it may be necessary to disable JWT validation in the receiving microservice.

!!! example "Example - Using docker-compose environment variable to disable secure JWT validation"
    ```yaml
    environment: 
      EDGEX_DISABLE_JWT_VALIDATION: "true"
    ```

Regardless of the setting of this variable, the API gateway
(and related security-proxy-auth microservice)
will always validate the incoming JWT.


#### EDGEX_STARTUP_DURATION

This environment variable sets the total duration in seconds allowed for the services to complete the bootstrap start-up. Default is 60 seconds.

!!! example "Example - Using docker-compose to set start-up duration to 120 seconds"
    ```yaml
    environment: 
      EDGEX_STARTUP_DURATION: "120"
    ```

#### EDGEX_STARTUP_INTERVAL

This environment variable sets the retry interval in seconds for the services retrying a failed action during the bootstrap start-up. Default is 1 second.

!!! example "Example - Using docker-compose to set start-up interval to 3 seconds"
    ```yaml
    environment: 
      EDGEX_STARTUP_INTERVAL: "3"
    ```

## Environment Overrides

There are two types of environment overrides which are `command-line` and `configuration`. 

!!! important
    Environment variable overrides have precedence over all command-line, local configuration and remote configuration. i.e. configuration setting changed in Consul will be overridden after the service loads the configuration from Consul if that setting has an environment override.

### Command-line Overrides

#### EDGEX_CONFIG_DIR

This environment variable overrides the [`-cd/--configDir` command-line option](../CommonCommandLineOptions/#confdir). 

!!! note
     All EdgeX service Docker images have this option set to `/res`.

!!! example "Example - Using docker-compose to override the configuration folder name"
    ```yaml
    environment: 
      EDGEX_CONF_DIR: "/my-config"
    ```

!!! edgey "EdgeX 3.0"
    The `EDGEX_CONF_DIR` environment variable is replaced by `EDGEX_CONFIG_DIR` in EdgeX 3.0.

#### EDGEX_CONFIG_FILE

This environment variable overrides the [`-cf/--configFile` command-line option](../CommonCommandLineOptions#file).

!!! example "Example - Using docker-compose to override the configuration file name used"
    ```yaml
    environment: 
      EDGEX_CONFIG_FILE: "my-config.yaml"
    ```

#### EDGEX_CONFIG_PROVIDER

This environment variable overrides the [`-cp/--configProvider` command-line option](../CommonCommandLineOptions#config-provider). 

Overriding with a value of `none` disables the use of the Configuration Provider.

!!! note
    All EdgeX service Docker images have this option set to `-cp=consul.http://edgex-core-consul:8500`.

!!! example "Example - Using docker-compose to override with different port number"
    ```yaml
    environment: 
      EDGEX_CONFIG_PROVIDER: "consul.http://edgex-consul:9500"
    
    or
    
    environment: 
      EDGEX_CONFIG_PROVIDER: "none"
    ```

!!! edgey "EdgeX 3.0"
    The `EDGEX_CONFIGURATION_PROVIDER` environment variable is replaced by `EDGEX_CONFIG_PROVIDER` in EdgeX 3.0.

#### EDGEX_COMMON_CONFIG

This environment variable overrides the [`-cc/--commonConfig` command-line option](../CommonCommandLineOptions#common-config).

!!! note
    The Common Config can only be specified when not using the Configuration Provider.

!!! example "Example - Override with a common configuration file at the command line"
    ```bash
    $ export EDGEX_COMMON_CONFIG=./my-common-configuration.yaml
    $ ./core-data
    ```

!!! edgey "EdgeX 3.0"
    The `EDGEX_COMMON_CONFIG` variable is new to EdgeX 3.0.
    

#### EDGEX_PROFILE

This environment variable overrides the [`-p/--profile` command-line option](../CommonCommandLineOptions#profile). When non-empty,  the value is used in the path to the configuration file. i.e. /res/my-profile/configuation.yaml.  This is useful when running multiple instances of a service such as App Service Configurable.

!!! example "Example - Using docker-compose to override the profile to use"
    ```yaml
    app-service-rules:
        image: edgexfoundry/docker-app-service-configurable:2.0.0
        environment: 
          EDGEX_PROFILE: "rules-engine"
        ...
    ```

This sets the `profile` so that the App Service Configurable uses the `rules-engine` configuration profile which resides at `/res/rules-engine/configuration.yaml`

#### EDGEX_USE_REGISTRY

This environment variable overrides the [`-r/--registry` command-line option](../CommonCommandLineOptions#registry). 

!!! note
    All EdgeX service Docker images have this option set to `--registry`.

!!! example "Example - Using docker-compose to override use of the Registry"
    ```yaml
    environment: 
      EDGEX_USE_REGISTRY: "false"
    ```

### Configuration Overrides

Any configuration setting from a service's `configuration.yaml` file can be overridden by environment variables. The environment variable names have the following format:

```
<SECTION-NAME>_<KEY-NAME>
<SECTION-NAME>_<SUB-SECTION-NAME>_<KEY-NAME>
```

!!! example "Example - Environment Overrides of Configuration"
    ``` yaml   
    CONFIG  : Writable:    
               LogLevel: "INFO"    
    ENVVAR : WRITABLE_LOGLEVEL=DEBUG    

    CONFIG : Clients:
               core-data:
                 Host: "localhost"
    ENVVAR : CLIENTS_CORE_DATA_HOST=edgex-core-data    
    ```    

### SecretStore Overrides

!!! edgey "EdgeX 3.0"
    For EdgeX 3.0 the **SecretStore** configuration has been removed from each service's configuration files. It now has default values which can be overridden with environment variables.

The environment variables overrides for **SecretStore** configuration remain the same as in 2.x releases. The following are SecretStore** fields that commonly need to be overridden.

- SECRETSTORE_HOST
- SECRETSTORE_RUNTIMETOKENPROVIDER_ENABLED
- SECRETSTORE_RUNTIMETOKENPROVIDER_HOST

The  complete list of **SecretStore** fields and defaults can be found in the file [here](https://github.com/edgexfoundry/go-mod-bootstrap/blob/main/config/types.go) (search for SecretStoreInfo). The defaults for the remaining fields typically do not need to be overridden, but may be overridden if needed using that same naming scheme as above.

### Notable Configuration Overrides

This section describes environment variable overrides that have special utility,
such as enabling a debug capability or facilitating code development.

#### TOKENFILEPROVIDER_DEFAULTTOKENTTL (security-secretstore-setup service)

This variable controls the TTL of the default secretstore tokens that are created for EdgeX microservices.
This variable defaults to `1h` (one hour) if unspecified.
It is often useful when developing a new microservice to set this value to a higher value, such as `12h`.
This higher value will allow the secret store token to remain valid long enough
for a developer to get a new microservice working and into a state where it can renew its own token.
(All secret store tokens in EdgeX expire if not renewed periodically.)