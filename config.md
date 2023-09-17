# Managing  Application Config in Rust
One of the tenets of a [Twelve-Factor app](https://12factor.net/) is to store application configs in the environment the application is deployed to. To pass config values to an application, developers typically use Environment Variables.  There a multiple ways to set these config values, we will look at three of them:
1. Reading from Environment Variables
2. `.env` files
3. Command line arguments

It is usually recommended to use the Rust type system to expres your data structures. We will represent configs out application needs in a struct:

```rs
#[derive(Debug, Default)]
pub struct Config {
    pub db_name: String,
    pub db_password: String,
    pub base_url: String,
    pub client_id: u32,
}
```
As configs can come from multiple sources, I prefer to have a `ConfigProvider` trait that can have multiple implementations depending on where the developer reads the configs from.

```rs
pub trait ConfigProvider {
    fn get_config(&self) -> &Config;
}
```

## 1: Reading from Environment Variables


## 2: Reading from Command Line Arguments


## 3: Reading from `.env` file
A common practice during development is to use a `.env` file holding the config values. You can have multiple env files according to the environment you are targeting, e.g `.env.development`, `.env.staging` and `.env.production`.
There is a nice crate,  [dotenv](https://crates.io/crates/dotenv) that handles the reading and parsing of these config files.

You define your `.env` file as follows:

```
DB_NAME=users
DB_PASSWORD=secret
BASE_URL=localhost:5432
CLIENT_ID=2
```

```rs
pub struct DotEnvConfigProvider(Config);

impl DotEnvConfigProvider {
    pub fn new() -> Self {
        use dotenv::dotenv;
        use std::env;
        dotenv().ok();
        let config = Config {
            db_name: env::var("DB_NAME").expect("Missing database name"),
            db_password: env::var("DB_PASSWORD").expect("Missing database password"),
            base_url: env::var("BASE_URL").expect("Missing base uri"),
            client_id: env::var("CLIENT_ID").trim().parse().expect("Missing client id"),
        };

        DotEnvConfigProvider(config)
    }
}

impl ConfigProvider for DotEnvConfigProvider {
    fn get_config(&self) -> &Config {
        &self.0
    }
}

impl Default for DotEnvConfigProvider {
    fn default() -> Self {
        Self::new()
    }
}
```

In your main function initialize the ConfigProvider and use it around in the application:

```rs
let config_provider = Box::new(DotEnvConfigProvider::new());
...
let db_name = &config_provider.get_config().db_name;
```