# Managing  Application Config in Rust
One of the tenets of a [Twelve-Factor app](https://12factor.net/) is to store application configs in the environment the application is deployed to. To pass config values to an application, developers typically use Environment Variables.  There a multiple ways to set these config values, we will look at three of them:
1. Command line arguments
2. Reading from Environment Variables
2. `.env` file

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

## 1: Reading from Command line Arguments
This the traditional of passing values to the binary's main function.

`cargo run mybinary -- arg1 arg2 KEY1=VALUE1 KEY2=VALUE2,VALUE3,VALUE4`

There is a nice crate [argmap](https://crates.io/crates/argmap) that can read the various command line argument formats and return as common Rust types that are easier to work with.

`cargo add argmap`

```rs
// main.rs
pub fn main(){
    let (args, argv) = argmap::parse(env::args());
    let cmd_config_provider = CmdConfigProvider::new(argv);
}
```
The parse method returns a tuple `(Vec<String>, HashMap<String, Vec<String>>`. The first Vec is a list of command line arguments passed without key-value pairs. The second value are arguments passed as key-value pairs.

```rs
// config.rs

pub struct CmdConfigProvider(Config);

impl CmdConfigProvider {
    pub fn new(args: Vec<String>, argv: HashMap<String, Vec<String>>) -> Self {
        let db_name = args.iter().nth(1).expect("Missing config");
        let db_password = args.iter().nth(2).expect("Missing config");
        let home_uri = argv.get("home_uri").expect("Missing config").to_vec().to_vec();
        let client_id = argv.get("client_id").expect("Missing config").to_vec().to_vec();
        let config = Config {
            db_name: db_name.to_string(),
            db_password: db_password.to_string(),
            home_uri: home_uri.first().expect("Missing config").to_string(),
            client_id: client_id.first().expect("Missing config").to_string(),
        };
        CmdConfigProvider(config)
    }
}

impl ConfigProvider for CmdConfigProvider {
    fn get_config(&self) -> &Config {
        &self.0
    }
}

impl Default for CmdConfigProvider {
    fn default() -> Self {
        Self::new(Vec::new(), HashMap::new())
    }
}
```

In your main function initialize the ConfigProvider and use it around in the application:

```rs
// main.rs

let config_provider = Box::new(CmdConfigProvider::new(args, argv));
...
let db_name = &config_provider.get_config().db_name;
```


## 2: Reading from Environment Variables

Here we read from the container/operating system environment variables. THere is no need of a crate as Rust standard library provides this feature out of the box

```rs
// main.rs
pub fn main(){
    let env_config_provider = EnvVarProvider::new(env::vars().collect());
}
```

```rs
// config.rs

pub struct EnvVarProvider(Config);

impl EnvVarProvider {
    pub fn new(args: HashMap<String, String>) -> Self {
        let config = Config {
            home_uri: args.get("HOME_URI").expect("Missing config").to_string(),
            db_name: args.get("DB_NAME").expect("Missing config").to_string(),
            db_password: args.get("DB_PASSWORD").expect("Missing config").to_string(),
            home_uri: args.get("HOME_URI").expect("Missing config").to_string(),
            client_id: args.get("CLEINT_ID").expect("Missing config").to_string()
        };
        EnvVarProvider(config)
    }
}

impl ConfigProvider for EnvVarProvider {
    fn get_config(&self) -> &Config {
        &self.0
    }
}

impl Default for EnvVarProvider {
    fn default() -> Self {
        Self::new(HashMap::new())
    }
}
```

Similarly, in our main function:

```rs
// main.rs

let config_provider = Box::new(EnvVarProvider::new());
...
let db_name = &config_provider.get_config().db_name;

```


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
// config.rs

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
// main.rs

let config_provider = Box::new(DotEnvConfigProvider::new());
...
let db_name = &config_provider.get_config().db_name;
```