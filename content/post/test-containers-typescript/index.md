---
title: Docker Tescontainers with TypeScript
description: A walkthrough of how to use Docker Testcontainers in any Typescript based project.
slug: test-containers-typescript
date: 2023-09-08 00:00:00+0000
image: cover.jpg
draft: true
categories: 
  - containers
  - development
  - typescript
  - docker
  - testcontainers
  - testing
---


## Keeping Front-End React Native App in Sync with Backend Changes

In the world of React Native development, staying in sync with backend changes is crucial. Unexpected backend shifts can sometimes throw the front end off track, causing a ripple of integration issues. In this post, we will explore a strategy that leverages modularity and integration testing to keep the two ends harmoniously in line.

### The Challenge:
The rapid evolution of backend technologies often means that changes occur dynamically. This can lead to:

1. Unexpected bugs or crashes on the frontend.
2. Incorrect data presentation due to altered API responses.
3. Other unforeseen integration complications.

### The Modular Solution:
Let's visualize a simple weather app developed in React Native. The app fetches the current weather for a given city. A user enters the city name, and the app displays the temperature, weather condition, and other relevant details.

To ensure our React Native application is resilient to backend changes, we'll design it in two modules:

1. **App Module**: Comprises the user interface, where users interact with the app.
2. **Backend Service Module**: Manages the API interactions, handling data requests, and responses.

Our key focus is on the **Backend Service Module**. By modularizing it, we can now test this module's integration with the backend service in isolation. This is a game-changer. It means we can verify our integration without spinning up the entire app with all its dependencies, ensuring a lightweight and effective testing strategy.

For our weather app, let's employ a mock server using the public Docker container `weather-api-mock` (a hypothetical mock version of a weather API). This will simulate our backend.

#### Backend Service Module in Action:
```typescript
// backend.service.module.ts (React Native)

export const BackendService = {
  async fetchWeather(city: string) {
    const response = await fetch(`http://weather-api-mock/service/${city}`);
    const data = await response.json();
    return data;
  },
};
```

And on the front end, our app module calls this service:

```typescript
// app.module.ts (React Native)

import { BackendService } from './backend.service.module';

export const WeatherComponent = () => {
  const [weatherData, setWeatherData] = useState(null);

  const fetchWeather = async (city) => {
    const data = await BackendService.fetchWeather(city);
    setWeatherData(data);
  };

  return (
    <View>
      <TextInput placeholder="Enter city" onSubmitEditing={fetchWeather} />
      <Text>{weatherData?.temperature}</Text>
      <Text>{weatherData?.condition}</Text>
    </View>
  );
};
```

Of course, let's dive right into the TypeScript code for the integration between the App and the backend module and the integration test using test containers.

### 1. **Backend Service Module**:
This module interacts with our mock weather service.

```typescript
// backend.service.module.ts

export const BackendService = {
  async fetchWeather(city: string) {
    const response = await fetch(`http://weather-api-mock/service/${city}`);
    const data = await response.json();
    return data;
  },
};
```

### 2. **App Module**:
The app module that calls the backend service module to fetch weather data:

```typescript
// app.module.ts

import { BackendService } from './backend.service.module';

export const WeatherComponent = () => {
  const [weatherData, setWeatherData] = useState(null);

  const fetchWeather = async (city) => {
    const data = await BackendService.fetchWeather(city);
    setWeatherData(data);
  };

  return (
    <View>
      <TextInput placeholder="Enter city" onSubmitEditing={fetchWeather} />
      <Text>{weatherData?.temperature}</Text>
      <Text>{weatherData?.condition}</Text>
    </View>
  );
};
```

### 3. **Integration Test using Test Containers**:

To write our test, we need to simulate our backend using test containers. Assuming there's a public Docker container named `weather-api-mock`, we can set up a Test Container and run integration tests against this mock server.

```typescript
// backend.service.module.test.ts

import { BackendService } from './backend.service.module';
import { GenericContainer } from "testcontainers";

describe('BackendService', () => {
  let container;
  let mockAPIUrl;

  beforeAll(async () => {
    container = await new GenericContainer('weather-api-mock').start();
    mockAPIUrl = `http://localhost:${container.getMappedPort(80)}/service`;
  });

  afterAll(async () => {
    await container.stop();
  });

  it('fetches weather data', async () => {
    const city = "London";
    const weather = await BackendService.fetchWeather(city);

    expect(weather).toHaveProperty('temperature');
    expect(weather).toHaveProperty('condition');
  });
});
```

This test first starts the `weather-api-mock` container, runs the test, and then stops the container. It checks whether the `fetchWeather` function from our `BackendService` can fetch the required data. 

Remember, you'll need to install the `testcontainers` library to run the above test and have Docker running on the machine where this test is executed.

### CI/CD: The Early Warning System:

Executing these tests on individual development machines can be tedious, given the Docker dependencies. Instead, the sweet spot is to incorporate these integration tests into a CI/CD pipeline.

With a CI/CD process, the pipeline automatically:

1. Pulls the latest `weather-api-mock` Docker image.
2. Runs the integration tests against the current frontend code.
3. Notifies of any discrepancies or failures.

Here's a simplified GitHub Actions setup to achieve this:

```yaml
name: Integration Test

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Pull latest weather API mock Docker image
      run: docker pull weather-api-mock:latest

    - name: Run integration tests
      run: npm test ./path-to/backend.service.module.test.ts
```

### Wrapping Up:
The modular approach, coupled with CI/CD, ensures that our React Native weather app remains harmonious with backend evolutions. This method provides an early warning mechanism, reducing integration headaches and guaranteeing a smoother development experience.
