### Problem Statement:  
the v3 api is now completely functional on diego when using buildpacks, we now wish to support docker apps in v3.

we've made some assumptions around buildpacks that don't fit for docker and are now looking for the proper abstractions to support buildpack, docker and unknown future app types

we would love some feedback on the approach we are considering


This affects many of our API endpoints, so I'll walk through them one at a time:

#### Packages
Would become a polymorphic resource, where the request/response would be based on a type.  Our only current type is bits, which requires a user
to upload.  We want to introduce docker registry as a source.  As a strawman, I will also include an imaginary git package.

##### Bits
Requires an upload before it is usable

###### Request
POST /v3/packages
```json
{
  "type": "bits"
}
```

###### Response
error: user friendly error message if the upload process fails

hash: a sha of the bits

```json
{
  "guid": "guid-68f784a1-50cf-4222-a1a7-84738d834a60",
  "state": "AWAITING_UPLOAD",

  "type": "bits",

  "error": null,   
  "hash": {
    "type": "sha1",
    "value": null
  }
}
```

##### Docker

###### Request
POST /v3/packages
```json
{
  "type": "docker",
  "image": "cloudfoundry/runtime-ci:latest",
  "credentials": {
    "user": "user name",
    "password": "s3cr3t",
    "email": "email@example.com",
    "login_server": "https://index.docker.io/v1/"
  },
}
```

###### Response
*Note: error and hash no longer seem to be relevant

```json
{
  "guid": "guid-68f784a1-50cf-4222-a1a7-84738d834a60",
  "state": "READY",

  "type": "docker",

  "image_path": "cloudfoundry/runtime-ci:latest",
  "credentials": {
    "user": "user name",
    "password": "s3cr3t",
    "email": "email@example.com",
    "login_server": "https://index.docker.io/v1/"
  }
}
```

##### Git

###### Request
POST /v3/packages
```json
{
  "type": "git",
  "url": "http://git.awesome.com:master",
  "credentials": {
    "user": "user",
    "password": "password",
    "api_key": "asdfasdfasdfasdf"
  }
}
```

###### Response
*Note: error and hash no longer seem to be relevant

```json
{
  "guid": "guid-68f784a1-50cf-4222-a1a7-84738d834a60",
  "state": "READY",

  "type": "git",

  "url": "http://git.awesome.com:master",
  "credentials": {
    "user": "user",
    "password": "password",
    "api_key": "asdfasdfasdfasdf"
  }
}
```

#### Droplets
At this point we would introduce a `lifecycle` object that is polymorphic based on predefined supported lifecycles.
We would move staging response data into a `staging_response` field and it's structure would be defined by a combination
of the lifecycle used and the package type.

##### Example Lifecycles

###### Buildpack
```json
"lifecycle": {
  "type": "buildpack",
  "data": {
    "buildpack": "requested buildpack name",
    "stack": "cflinuxfs2"
  }
}
```

###### Docker
```json
"lifecycle": {
  "type": "docker",
  "data": {
    "use_caching": true
  }
}
```

##### Droplet from bits with Buildpack Lifecycle

###### Request
POST /v3/packages/:guid/droplets
```json
{
  "environment_variables": {
    "cloud": "foundry"
  },

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "name",
      "stack": "cflinuxfs2"
    }
  }
}
```

###### Response
```json
{
  "guid": "guid-c6e5009d-d984-457e-91a1-7cfb48f10a36",
  "state": "STAGED",
  "error": "example error",

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "requested buildpack name",
      "stack": "cflinuxfs2"
    }
  },
  "environment_variables": {
    "cloud": "foundry"
  },

  "staging_response": {
    "hash": {
      "type": "sha1",
      "value": "234-3434-3434"
    },
    "buildpack": "detected-buildpack",
    "stack": "cflinuxfs2",
    "process_types": {
      "web": "start",
      "worker": "start worker"
    }
  }
}
```

##### Droplet from git with Buildpack Lifecycle
Basically the same as from bits, but the staging_response contains some extra info (the git commit sha)

###### Request
POST /v3/packages/:guid/droplets
```json
{
  "environment_variables": {
    "cloud": "foundry"
  },

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "name",
      "stack": "cflinuxfs2"
    }
  }
}
```

###### Response
```json
{
  "guid": "guid-c6e5009d-d984-457e-91a1-7cfb48f10a36",
  "state": "STAGED",
  "error": "example error",

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "requested buildpack name",
      "stack": "cflinuxfs2"
    }
  },
  "environment_variables": {
    "cloud": "foundry"
  },

  "staging_response": {
    "hash": {
      "type": "sha1",
      "value": "234-3434-3434"
    },
    "buildpack": "detected-buildpack",
    "stack": "cflinuxfs2",
    "process_types": {
      "web": "start",
      "worker": "start worker"
    },
    "commit": "shathingy"
  }
}
```

##### Droplet from docker with Docker Lifecycle

###### Request
POST /v3/packages/:guid/droplets
```json
{
  "environment_variables": {
    "cloud": "foundry"
  },

  "lifecycle": {
    "type": "docker",
    "data": {
      "use_caching": true
    }
  }
}
```

###### Response
```json
{
  "guid": "guid-c6e5009d-d984-457e-91a1-7cfb48f10a36",
  "state": "STAGED",
  "error": "example error",

  "lifecycle": {
    "type": "docker",
    "data": {
      "use_caching": true
    }
  },
  "environment_variables": {
    "cloud": "foundry"
  },

  "staging_response": {
    "start_command": "some command",
    "image_path": "source_repo/sweetapp:latest",
    "cached": true
  }
}
```

#### Apps
Apps would gain an optional `lifecycle` object.  Any defaults to be used for the app could be defined here.  We would default
to a lifecycle for buildpacks with auto-detection.  These defaults could be overriden through the post body of the 
staging request (POST for droplets).


###### Request
POST /v3/apps
```json
{
  "name": "my_app",

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "name-363"
    }
  },

  "environment_variables": {
    "unicorn": "horn"
  },
  
  "relationships": {
    "space": { "guid": "the-guid" }
  }
}

```

###### Response
```json
{
  "guid": "guid-fcfb555c-3825-4c43-a52e-6562fdf3c097",
  "name": "my_app",
  "desired_state": "STOPPED",

  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "name-363",
      "stack": null,
    }
  },

  "environment_variables": {
    "unicorn": "horn"
  },

  "created_at": "2015-08-31T21:49:58Z",
  "updated_at": null
}

```

##### Modifying lifecycle
We should choose one of:

Root resource:

PATCH /v3/apps/guid
```json
{
  "lifecycle": {
    "type": "buildpack",
    "data": {
      "buildpack": "different buildpack"
    }
  }     
}
```

Or through a sub-resource:

PATCH /v3/apps/guid/lifecycle
```json
{
  "type": "buildpack",
  "data": {
    "buildpack": "different buildpack"
  }
}
```
