# Cloud Foundry MySQL Service

### Table of contents

[Differences from master Cloud Foundry release](#differences)

[Components](#components)

[Development](#development)

[Release notes & known issues](#release-notes)


<a name='differences'></a>
## Differences from master Cloud Foundry release

Check the jobs spec file for differences with master. The main difference is that to allow for cross DC clusters you need to provide a list of all IP's in the cluster prior to deployment. As it currently stands, this cannot be done dynamically.

You also need to provide a list of arbitrator nodes for use in the arbitrator job. This list is appended to the data node cluster_ips array within the mysql config.

As we have 2 BOSH directors, as it currently stands, you need to deploy one at a time to each director. The nodes will automatically join the cluster.

wsrep_cluster_address_string must be provided in your deployment and must be whitespace free due to how the MySQL config is parsed. This could probably be done in the release itself and create a whitespace free list from cluster_ips and arbitrator_node.

<a name='components'></a>
## Components

A BOSH release of a MySQL database-as-a-service for Cloud Foundry using [MariaDB Galera Cluster](https://mariadb.com/kb/en/mariadb/documentation/replication-cluster-multi-master/galera/what-is-mariadb-galera-cluster/) and a [v2 Service Broker](http://docs.cloudfoundry.org/services/).


<a name='proxy'></a>
### Proxy

Traffic to the MySQL cluster is routed through one or more proxy nodes. The current proxy implementation is [Switchboard](https://github.com/cloudfoundry-incubator/switchboard). This proxy acts as an intermediary between the client and the MySQL server, providing failover between MySQL nodes. The number of nodes is configured by the proxy job instance count in the deployment manifest.

**NOTE:** If the number of proxy nodes is set to zero, apps will be bound to the IP address of the first MySQL node in the cluster. If that IP address should change for any reason (e.g. loss of a VM) or a proxy was subsequently added, one would need to re-bind all apps to the IP address of the new node.

For more details see the [proxy documentation](/docs/proxy.md).

<a name="dashboard"></a>
### Dashboard

A user-facing service dashboard is provided by the service broker that displays storage utilization information for each service instance.
The dashboard is accessible by users via Single Sign-On (SSO) once authenticated with Cloud Foundry.
The dashboard URL can be found by running `cf service MY_SERVICE_INSTANCE`.

Service authors interested in implementing a service dashboard accessible via SSO can follow documentation for [Dashboard SSO](http://docs.cloudfoundry.org/services/dashboard-sso.html).

#### Prerequisites

1. SSO is initiated when a user navigates to the URL found in the `dashboard_url` field. This value is returned to cloud controller by the broker in response to a provision request, and is exposed in the cloud controller API for the service instance. A users client must expose this field as a link, or it can be obtained via curl (`cf curl /v2/service_instances/:guid`) and copied into a browser.

1. SSO requires the following OAuth client to be configured in cf-release. This client is responsible for creating the OAuth client for the MySQL dashboard. Without this client configured in cf-release, the MySQL dashboard will not be accessible but the service will be otherwise functional. Registering the broker will display a warning to this effect.

    ```yaml
    properties:
      uaa:
        clients:
          cc-service-dashboards:
            secret: cc-broker-secret
            scope: cloud_controller.write,openid,cloud_controller.read,cloud_controller_service_permissions.read
            authorities: clients.read,clients.write,clients.admin
            authorized-grant-types: client_credentials
    ```

1. SSO was implemented in v169 of cf-release; if you are on an older version of cf-release you'll encounter an error when you register the service broker. If upgrading cf-release is not an option, try removing the following lines from the cf-mysql-release manifest and redeploy.

    ```yaml
    dashboard_client:
      id: p-mysql
      secret: yoursecret
    ```

#### Implementation Notes

The following links show how this release implements [Dashboard SSO](http://docs.cloudfoundry.org/services/dashboard-sso.html) integration.

1. Update the broker catalog with the dashboard client [properties](https://github.com/cloudfoundry/cf-mysql-broker/blob/master/config/settings.yml#L26)
2. Implement oauth [workflow](https://github.com/cloudfoundry/cf-mysql-broker/blob/master/config/initializers/omniauth.rb) with the [omniauth-uaa-oauth2 gem](https://github.com/cloudfoundry/omniauth-uaa-oauth2)
3. [Use](https://github.com/cloudfoundry/cf-mysql-broker/blob/master/lib/uaa_session.rb) the [cf-uaa-lib gem](https://github.com/cloudfoundry/cf-uaa-lib) to get a valid access token and request permissions on the instance
4. Before showing the user the dashboard, [the broker checks](https://github.com/cloudfoundry/cf-mysql-broker/blob/master/app/controllers/manage/instances_controller.rb#L7) to see if the user is logged-in and has permissions to view the usage details of the instance.

### Broker Configuration

#### Require HTTPS when visiting Dashboard

The dashboard URL defaults to using the `https` scheme. This means any requests using `http` will automatically be redirected to `https` instead.
To override this, you can change `jobs.cf-mysql-broker.ssl_enabled` to `false`.

Keep in mind that changing the `ssl_enabled` setting for an existing broker will not update previously advertised dashboard URLs.
Visiting the old URL may fail if you are using the [SSO integration](http://docs.cloudfoundry.org/services/dashboard-sso.html),
because the OAuth2 client registered with UAA will expect users to both come from and return to a URI using the scheme
implied by the `ssl_enabled` setting.

Note:
If using `https`, the broker must be reached through an SSL termination proxy.
Connecting to the broker directly on `https` will result in a `port 443: Connection refused` error.

#### Trust Self-Signed SSL Certificates

By default, the broker will not trust a self-signed SSL certificate when communicating with cf-release.
To trust self-signed SSL certificates, you can change `jobs.cf-mysql-broker.skip_ssl_validation` to `true`.


<a name='development'></a>
## Development

See our [contributing docs](CONTRIBUTING.md) for instructions on how to make a pull request.

This BOSH release doubles as a `$GOPATH`. It will automatically be set up for
you if you have [direnv](http://direnv.net) installed.

    # fetch release repo
    mkdir -p ~/workspace
    cd ~/workspace
    git clone https://github.com/cloudfoundry/cf-mysql-release.git
    cd cf-mysql-release/

    # switch to develop branch (not master!)
    git checkout develop

    # automate $GOPATH and $PATH setup
    direnv allow

    # initialize and sync submodules
    ./scripts/update

If you do not wish to use direnv, you can simply `source` the `.envrc` file in the root
of the release repo.  You may manually need to update your `$GOPATH` and `$PATH` variables
as you switch in and out of the directory.

<a name='release-notes'></a>
## Release Notes, Known Issues, and Documentation

[Release Notes](https://github.com/cloudfoundry/cf-mysql-release/releases/)

[Known Issues](/docs/known-issues.md)

For more information, check out the [documentation](https://github.com/cloudfoundry/cf-mysql-release/tree/master/docs).

<a name='deploying'></a>
## Deploying

See https://github.com/cloudfoundry/cf-mysql-deployment to deploy cf-mysql release.
