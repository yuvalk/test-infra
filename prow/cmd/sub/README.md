# Sub

Sub is a Prow Cloud Pub/Sub adapter for handling CI Pub/Sub notification requests.
It currently supports Periodic Prow Job. Note that the prow job need to be defined in the configuration.

## Deployment Usage

Sub supports 2 running modes, Push and Pull. Note that each subscription need to be one or the other, but Sub will run both.

When deploy the sub component, you need to specify `--config-path` to your prow config, and optionally
`--job-config-path` to your prowjob config if you have split them up.


Options:
* `--push-secret-file`: Path to Pub/Sub Push secret file.
* `--dry-run`: Dry run for testing. Uses API tokens but does not mutate.
* `--grace-period`: On shutdown, try to handle remaining events for the specified duration.

### Push Server

In order to use the Push Mode, an HTTPs server needs to be setup and the URL must be defined in the Cloud Pub/Sub subscription.
More information at https://cloud.google.com/pubsub/docs/quickstart-console.

To secure even more requests from Cloud Pub/Sub, you can use `--push-secret-file` option.
When using a push secret add the token to the URL like so  https://myapp.mydomain.com/myhandler?token=application-secret.
More info at https://cloud.google.com/pubsub/docs/faq#security.

### Pull Server

All pull subscriptions need to be defined in Prow Configuration:

```
pubsub_subscriptions:
  "gcp-project-01":
  - "subscription-01"
  - "subscription-02"
  - "subscription-03"
  "gcp-project-02":
  - "subscription-01"
  - "subscription-02"
  - "subscription-03"
```

Sub must be running with `GOOGLE_APPLICATION_CREDENTIALS` environment variable pointing to the service
account credentials JSON file. The service account used must have the right permission on the
subscriptions (`Pub/Sub Subscriber`, and `Pub/Sub Editor`).

More information at https://cloud.google.com/pubsub/docs/access-control.

### Sending a Pub/Sub Notification

#### Periodic Prow Jobs

When creating your Pub/Sub message, add an attributes with key ```prow.k8s.io/pubsub.EventType```
and value ```prow.k8s.io/pubsub.PeriodicProwJobEvent```, and a payload like so:

```yaml
{
  "name":"my-periodic-job",
  "envs":{
    "GIT_BRANCH":"v.1.2",
    "MY_ENV":"overwrite"
  },
  "labels":{
    "myLabel":"myValue",
  },
  "annotations":{
    # GCP project where prowjobs statues are published by prow. Must also provide "prow.k8s.io/pubsub.topic" to take effect.
    # It's highly recommended to configure this even if prowjobs are monitorings by other means, since this is also where errors are
    # reported when the job failed to be triggered
    "prow.k8s.io/pubsub.project":"myProject",
    "prow.k8s.io/pubsub.runID":"asdfasdfasdf",
    # GCP pubsub topic where prowjobs statues are published by prow, must be a different topic from where this payload is published to
    "prow.k8s.io/pubsub.topic":"myTopic"
  }
}
```

This will find and start the periodic job ```my-periodic-job```, and add / overwrite the
annotations and envs to the Prow job. The ```prow.k8s.io/pubsub.*``` annotations are
used to publish job status.

_Note: periodic jobs always clone source code from ref instead of specific SHA, if it's desired to trigger a prowjob on specific SHA you can use [postsubmit job](#postsubmit-prow-jobs)_

#### Presubmit Prow Jobs

Triggering presubmit job is similar to periodic jobs. Two things to change:

- instead of an attributes with key ```prow.k8s.io/pubsub.EventType``` and value
```prow.k8s.io/pubsub.PeriodicProwJobEvent```, replace the value with ```prow.k8s.io/pubsub.PresubmitProwJobEvent```
- requires setting `refs` instructing presubmit jobs how to clone source code:

```yaml
{
  # Common fields as above
  "name":"my-presubmit-job",
  "envs":{...},
  "labels":{...},
  "annotations":{...},

  "refs":{
    "org": "org-a",
    "repo": "repo-b",
    "base_ref": "main",
    "base_sha": "abc123",
    "pulls": [
      {
        "sha": "def456"
      }
    ]
  }
}
```

This will start presubmit job ```my-presubmit-job```, clones source code like pull requests
defined under ```pulls```, which merges to ```base_ref``` at ```base_sha```.

(There are more fields can be supplied, see [full documentation](https://github.com/kubernetes/test-infra/blob/18678b3b8f4bc7c51475f41964927ff7e635f3b9/prow/apis/prowjobs/v1/types.go#L883). For example, if you want the job to be reported on the PR, add ```number``` field right next to ```sha```)

#### Postsubmit Prow Jobs

Triggering presubmit job is similar to periodic jobs. Two things to change:

- instead of an attributes with key ```prow.k8s.io/pubsub.EventType``` and value
```prow.k8s.io/pubsub.PeriodicProwJobEvent```, replace the value with ```prow.k8s.io/pubsub.PostsubmitProwJobEvent```
- requires setting `refs` instructing postsubmit jobs how to clone source code:

```yaml
{
  # Common fields as above
  "name":"my-postsubmit-job",
  "envs":{...},
  "labels":{...},
  "annotations":{...},

  "refs":{
    "org": "org-a",
    "repo": "repo-b",
    "base_ref": "main",
    "base_sha": "abc123"
  }
}
```

This will start postsubmit job ```my-postsubmit-job```, clones source code from ```base_ref```
at ```base_sha```.

(There are more fields can be supplied, see [full documentation](https://github.com/kubernetes/test-infra/blob/18678b3b8f4bc7c51475f41964927ff7e635f3b9/prow/apis/prowjobs/v1/types.go#L883))
