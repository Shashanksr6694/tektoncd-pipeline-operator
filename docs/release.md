## Building Operator Bundle

1. Sync latest master
1. new branch
1. Sync the downstream pipeline repo
1. Checkout release branch
1. Copy the release yaml from the pipelines repo to operator
   deploy/resources/<version>
1. update operator version pkg/flag/flag.go
1. test the operator using `up local`
    ```
    make opo-test-e2e-up-local
    ```
1. Build operator image and test operator deployment
    build image, push image and update deployment manifest
    ```
    make opo-build-push-update VERSION=0.9.2
    ```
    test operator deployment
    ```
    make opo-test-e2e
    ```
    
1. make CSV (make sure that the project base directory name is `openshift-pipelines-operator`)

    - `VERSION`: version of current release
    - `FROM_VERSION`: previous CSV version from which CSV metadata should be copied
    - `CHANNEL`: targeted channel
      ```
      make opo-new-csv VERSION=0.9.2 FROM_VERSION=0.8.2 CHANNEL=canary 
      ```
  
    You might  need  to edit the package.yaml to remove any duplicate channels. 
    Ensure that the currentCSV and channel names are as expected
    
    e.g.
      ```
      channels:
      - currentCSV: openshift-pipelines-operator.v0.5.0
        name: dev-preview
      - currentCSV: openshift-pipelines-operator.v0.7.0
        name: dev-preview
      ```

    will need to be corrected to:

    ```
    channels:
    - currentCSV: openshift-pipelines-operator.v0.7.0
      name: dev-preview
    ```

    See existing package in community operators for reference


1. verify operator bundle (deploy/olm-catalog/openshift-pipelines-operator directory)
    ```
    operator-courier verify --ui_validate_io \
      deploy/olm-catalog/openshift-pipelines-operator
    ```

## Test Operator Bundle on OLM

1. operator-courier use that to push the app bundle

    NOTE:
    
    You can obtain quay token by running `./scripts/get-quay-token` in
    operator-courier repo. see [Push to quay.io](https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md#push-to-quayio)

    ```
     make opo-push-quay-app VERSION=0.9.2 TOKEN=$TOKEN QUAY_NAMESPACE=nikhilthomas
    ```
    **NOTE** : special characters in password created issues when courier tried to
    push the app bundle.

1. Ensure that the application in quay is public


1. Create an operator source for the app bundle .e.g

    ```
    apiVersion: operators.coreos.com/v1
    kind: OperatorSource
    metadata:
      name: <QUAY_USER>-operators
      namespace: openshift-marketplace
    spec:
      type: appregistry
      endpoint: https://quay.io/cnr
      registryNamespace: <QUAY_USER>
      displayName: "<QUAY_USER> Operators"
      publisher: "<Your Name>"
    ```
    see: [Testing deployment on OpenShift](https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md#testing-operator-deployment-on-openshift)

    Validate operator source by
    ```
    oc get operatorsource <QUAY_USER>-operators -n openshift-marketplace -o yaml
    oc get catalogsources <QUAY_USER>-operators -n openshift-marketplace -o yaml
    
    ```

    Should see "Success: True" or something like that


    1. Create a subscription to install operator in `openshift-operators` ns
    ```
    
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: <QUAY_USER>-pipelines-subsription
      namespace: openshift-operators
    spec:
      channel: dev-preview
      name: openshift-pipelines-operator
      source: <QUAY_USER>-operators
      sourceNamespace: openshift-marketplace
    
    ```

    1. Run scorecard against the generated CSV
    
    ```
    operator-sdk scorecard \
      --olm-deployed \
      --csv-path deploy/olm-catalog/openshift-pipelines-operator/0.7.0/openshift-pipelines-operator.v0.7.0.clusterserviceversion.yaml \
      --namespace openshift-operators \
      --cr-manifest ./deploy/crds/operator_v1alpha1_config_cr.yaml \
      --crds-dir ./deploy/crds/
    
    ```

see: [testing with scorecard](https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md#testing-with-scorecard)

1. Publish to community operators
```
cp tmp/openshift-pipelines-operator/* \
  <community-git-repo>/community-operators/openshift-pipelines-operator
```

1. Submit a PR: e.g: https://github.com/operator-framework/community-operators/pull/756

see [Publishing your operator](https://github.com/operator-framework/community-operators/blob/master/docs/contributing.md#package-your-operator)
