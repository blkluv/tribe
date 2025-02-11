# OpenTogetherTube

[![CI/CD](https://github.com/dyc3/opentogethertube/actions/workflows/main.yml/badge.svg)](https://github.com/dyc3/opentogethertube/actions/workflows/main.yml)
[![codecov](https://codecov.io/gh/dyc3/opentogethertube/branch/master/graph/badge.svg)](https://codecov.io/gh/dyc3/opentogethertube)
[![Docker size](https://img.shields.io/docker/image-size/dyc3/opentogethertube)](https://hub.docker.com/r/dyc3/opentogethertube)

The easy way to watch videos with your friends.

https://opentogethertube.com/

# Deployment

See the [deployment docs](docs/how-to-deploy.md).

# Contributing

Contributions are welcome! Check out issues that have the "good first issue" label.

## Setting up your dev environment

### Prerequisites

This project targets node 18 and up.

### Setup

1. Fork this repo and clone it.
2. In a terminal, navigate to the `opentogethertube` folder and run
```
npm install -g yarn
yarn
```
3. Next you need to set up your configuration. Start by copying the example
config in the `env` folder to a new file called `development.toml`
```
cp env/example.toml env/development.toml
```
4. Create a new project on [Google Cloud](https://console.cloud.google.com)
5. Add "YouTube Data API v3" and "Google Drive API" to the project
6. Obtain a YouTube API key
7. Obtain a Google Drive API key
	- _Not necessary if you don't plan to stream videos from Google Drive, which you probably shouldn't do anyway because Google doesn't like that._
8. Open `env/development.toml` and put in the appropriate api keys.
9. Initialize your local database.
```
yarn workspace ott-server run sequelize-cli db:migrate
```
10.  Install [redis](https://redis.io). This is used to store room state and user sessions across server restarts.

## Testing

To run the unit test suite, run
```
yarn test
```

To run the e2e component test suite, run
```
yarn run cy:run --component
```

To run the e2e test suite, run
```
yarn run cy:run
```

However, while you're developing, you'll probably want to run the tests in headed mode. To do this, run
```
yarn run cy:open
```
This works for both the component and e2e tests.

## How to run

This project has 2 main components: the client and the server. You can run
both of them simultaneously using the command
#### Linux / Mac
```
yarn run dev
```
#### Windows
```
yarn run dev-windows
```

Sometimes, you may want to run them seperately so you can use breakpoints to
debug. Using VSCode, this is trivial.

To start the server: `Debug > Select "Launch Program" > Start`

To start the client: `yarn workspace ott-client serve`
