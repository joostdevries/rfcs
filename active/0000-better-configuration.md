- Start Date: 2015-08-06
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Summary

Let config/environment.js control build options normally set in `ember-cli-build.js` and improve configuration logic.

# Motivation

The current method of configuring is quite complex but lacks support for some faily basic use-cases. It is different from the rest of someone's app (not an ES6 module) but doesn't cover build parameters (these are configured in ember-cli-build.js). It also lacks a proper method of separating what is included in the client-side config and what is stripped out.

I propose a simpler system which covers all configuration relevant to both build, environment and app. This should also make adding additional environments (besides the default `production`, `development` and `test`) easier.

Related issues:

- #3457
- #2860
- #2489
- #2481
- #2135

# Detailed design

The initial draft for an implementation was made by @rwjblue in https://gist.github.com/rwjblue/ecb483b15da77582d0b9. It details a pattern of inheritance where one of the root configurationa is imported from `ember-cli` and an App can extend this root configuration to create an infinite amount of configurations. Each configuration is in a file with `environment-name` as name (eg `staging.js`, `production.js`).

## Basic example

The `Environment` class extends from `CoreObject`. Ember CLI provides four `Evironments`: the base class, production, development and test.

```javascript
// config/environments/production.js
var ProductionEnvironment = require('ember-cli/environment/production');

module.exports = ProductionEnvironment.extend({
  // Object describing the initial runtime config
  runtimeConfig: {}
  // Object describing the initial build config (passed to EmberApp)
  buildConfig: {}
  // Object describing the initial server config
  serverConfig: {}

  // Hook to read/override the runtime config
  runtime: function(config) {}
  // Hook to read/override the build config
  build: function(config) {}
  // Hook to read/override the server config
  server: function(config) {}
  // Hook to read/override the build config
  brocolli: function(app, config) {}
});
```

### Runtime configuration

```javascript
// config/environments/production.js
var ProductionEnvironment = require('ember-cli/environment/production');

module.exports = ProductionEnvironment.extend({
  runtimeConfig: {
    modulePrefix: 'some-app',
    baseURL: '/',
    locationType: 'auto',
    EmberENV: {
      FEATURES: {
        // Here you can enable experimental features on an ember canary build
        // e.g. 'with-controller': true
      }
    },

    APP: {
      // Here you can pass flags/options to your application instance
      // when it is created
    }
  }
});
```

### Build configuration

```javascript
// config/environments/production.js
var ProductionEnvironment = require('ember-cli/environment/production');

module.exports = ProductionEnvironment.extend({
  buildConfig: {
    fingerprint: {
      enabled: true,
      prepend: 'https://subdomain.cloudfront.net/'
    },
    sourcemaps: {
      enabled: false,
    },
    minifyCSS: { enabled: true },
    minifyJS: { enabled: true },

    tests: false,
    hinting: false,

    vendorFiles: {
      'handlebars.js': {
        staging:  'bower_components/handlebars/handlebars.runtime.js'
      },
      'ember.js': {
        staging:  'bower_components/ember/ember.prod.js'
      }
    }
  }
});
```

### Brocolli

```javascript
// config/environments/production.js
var ProductionEnvironment = require('ember-cli/environment/production');

module.exports = ProductionEnvironment.extend({
  brocolli: function(app, config) {
    app.import('vendor/some-asset.js');
  }
});
```

## Addons and overrides

Addons often need to read certain configuration values and sometimes also need
to write back to the configuration. These cases can involve any kind of configuration (build, runtime etc.).

To facilitate this, addons can extend the `Environment` provided by the app and use
hooks to read the configuration and apply overrides.

```javascript
// some-addon/config/production.js
var ProductionEnvironment = require('ember-cli/environment/production');

module.exports = ProductionEnvironment.extend({
  runtime: function(config) {
    this._super.apply(this, arguments);
    config.set('torii.providers', ['google', 'facebook']);
  }
});
```

## Mixins

```javascript
// config/environments/development.js
var DevelopmentEnvironment = require('ember-cli/environment/development');
var BaseConfig = require('./base');

module.exports = DevelopmentEnvironment.extend(BaseConfig, {
  // Application configuration here
});
```

## Implementation in Project

```javascript
Project.prototype.getAddonsConfig = function(environment) {
  this.initializeAddons();

  return this.addons.reduce(function(config, addon) {
    if (addon.config) {
      config = config.extend(addon.config);
    }

    return config;
  }, environment);
};
```

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?