# Securing Elasticsearch with Shield [](id=securing-elasticsearch-with-shield)

Elasticsearch makes storing, searching, and analyzing your Liferay search data
easy. When it comes to securing that data, use Elasticsearch's [Shield
plugin](https://www.elastic.co/guide/en/shield/2.2/index.html). To use Shield
with @product@, you need the *Enterprise Search-Standard* subscription.  This
subscription gives you access to an adapter plugin for configuring Liferay for
Shield. 

With Shield you can prevent unauthorized users from accessing the Elasticsearch
cluster, preserve data integrity, and create an audit trail to inspect
suspicious activity. This guide shows you the basics of how to install and
configure Shield, and then how to configure Liferay for Shield, using a
convenient Shield adapter plugin.

+$$$

**Note:** The Shield plugin can only be used when you're running Elasticsearch
in *remote mode*. If you're not sure what that means refer to the [Configuring
Elasticsearch article](discover/deployment/-/knowledge_base/7-0/configuring-elasticsearch).
It's not possible to install Shield into Liferay's default embedded
Elasticsearch--and you shouldn't be using embedded Elasticsearch in production
anyway.

$$$

Here's the process for configuring Shield:

-  Install the Shield plugin on Elasticsearch
-  Create a user for Liferay, with user name and password.
-  Install your [Shield license](https://www.elastic.co/guide/en/shield/2.2/license-management.html).
-  Install Liferay's Shield adapter plugin and configure it.
-  Enable Transport Layer Security (TLS) to encrypt your connection between
    Liferay and Elasticsearch.

These terms will be useful to understand as you read this guide:

-  *Elasticsearch Home* refers to the root folder of your unzipped Elasticsearch
  installation (for example, `elasticsearch-2.2.0`).
-  *Liferay Home* refers to the root folder of your Liferay installation. It 
  contains the `osgi`, `deploy`, `data`, and `license` folders.

## Installing Shield on Elasticsearch [](id=installing-shield-on-elasticsearch)

First install the Shield plugin on your Elasticsearch cluster.

1. Navigate to Elasticsearch Home and install the license plugin and the Shield
   plugin by executing

        ./bin/plugin install license

    and then

        ./bin/plugin install shield

2. Next, you need to prepare for Shield to authenticate requests. 

    Users making requests to an Elasticsearch installation protected by Shield
    need to be part of the *realm*, a user database configured for Shield. You
    can use the native user management system built into Shield, called
    *esusers*, or you can use an external system like LDAP.

    *Roles* for Shield are defined in `[Elasticsearch_Home]/shield/roles.yml`
    and include the following:

    `admin`: Has permission to perform any cluster or index action.

    `power_user`: Has permission to monitor the cluster and perform any index
    action.

    `user`: Has permission to perform read actions on any index.

    So who is the user you need to configure for Liferay? It's Liferay itself,
    and it needs the `admin` role. Liferay's Elasticsearch client sends its
    authentication token (in other words, it's user name and password) to
    Shield. Since Shield also has the authentication token stored in its user
    database, Liferay is a recognized user and has no problems communicating
    with the Elasticsearch cluster.

    From Elasticsearch Home, add an `admin` user named *liferay* to the esusers
    database:

        ./bin/shield/esusers useradd liferay -r admin

    When prompted, enter the password *liferay*.

    Note: Of course you can change these values if desired. Consider naming the
    user *shieldy_mcshieldface*, for example.<sup>[1](#footnote1)</sup>

3. Start Elasticsearch.

        ./bin/elasticsearch

4. Install your license file:

        curl -XPUT -u liferay 'http://localhost:9200/license?acknowledge=true' -d @license.json

    Enter the password you configured, and you'll get a confirmation message
    that looks like this:

        {"acknowledged":true}

5. To test that you have access to Elasticsearch, enter

        curl -u liferay:liferay -XGET 'http://localhost:9200/'

    and you'll see the Elasticsearch node information printed in the console:

        {
          "name" : "Amphibius",
          "cluster_name" : "LiferayElasticsearchCluster",
          "version" : {
            "number" : "2.2.0",
            "build_hash" : "8ff36d139e16f8720f2947ef62c8167a888992fe",
            "build_timestamp" : "2016-01-27T13:32:39Z",
            "build_snapshot" : false,
            "lucene_version" : "5.4.1"
          },
          "tagline" : "You Know, for Search"

For more information on installing Shield, see the [Elasticsearch
documentation](https://www.elastic.co/guide/en/shield/2.2/installing-shield.html).

Once Shield is installed, you can configure Liferay's Shield adapter.

## Installing and Configuring Liferay's Shield Adapter [](id=installing-and-configuring-liferays-shield-adapter)

On the Liferay side of the equation, you need to configure the authentication
token for the *liferay* Shield user you created in the previous section.
Liferay has a Shield adapter plugin for this purpose. 

First install the Shield adapter plugin (called *Liferay Portal Search
Elasticsearch Shield*). Once the plugin is installed, there's a new Shield
Configuration entry in the System Settings application (*Control Panel* &rarr;
*Configuration* &rarr; *System Settings*), under the Foundation heading.
Configure it so that its user name and password match the *liferay* user you
added to Shield.

You can configure the Shield adapter in the System Settings section in the
Control panel, or through an OSGi configuration file. 

Follow these steps to configure the Shield adapter using an OSGi configuration
file:

1. Create a file named `com.liferay.portal.search.elasticsearch.shield.configuration.ShieldConfiguration.cfg` in `[Liferay_Home]/osgi/configs`.

2. Add this content:

        password=liferay
        requiresAuthentication=true
        requiresSSL=false
        username=liferay

    For a description of all the Shield adapter's configuration options, see the
    section *Available Shield Adapter Configurations*.

3. Start Liferay.

Follow these steps to configure the Shield adapter using System Settings:

1. Navigate to *Control Panel* &rarr; *Configuration* &rarr; *System Settings*,
   and click on the *Foundation* heading. Navigate to, or search for, the
   *Shield Configuration* entry and click on it.

2. Set the password to *liferay*, check the *Requires authentication* box,
   uncheck the *Requires SSL* box, and make sure the user name is *liferay*.
   Click *Update* when you're done.

   Note: If you set a different user name and password while configuring Shield,
   make sure those match the user name and password you configure here.

    ![Figure 1: You can configure the Liferay Portal Search Elasticsearch Shield plugin from System Settings.](../../../images-dxp/shield-adapter-settings.png)

3. Now you can re-index against Elasticsearch, and your data is secured by
   Shield. To re-index, go to the Control Panel's Configuration section and click
   *Server Administration*. Find the Index Actions heading and click *Execute*
   next to *Reindex all search indexes.*

For a complete list of the Shield adapter's available configuration options, see
[here](discover/reference/-/knowledge_base/7-0/shield-settings)

## Encrypting Elasticsearch Connections [](id=encrypting-elasticsearch-connections)

It's great to ensure that your Elasticsearch connection is authenticated, but
right now the authentication token is sent in plain text. For additional
security you can enable Transport Layer Security (TLS) encryption.

This configuration uses a self signed certificate. See the [Elasticsearch
documentation](https://www.elastic.co/guide/en/shield/2.2/ssl-tls.html) for
alternative configuration approaches.

Both Elasticsearch and Liferay (the client) use the same keystore file in
this scenario.

1. Stop Liferay and Elasticsearch.
2. To generate a self signed certificate and key, navigate to
   `[Elasticsearch_Home]/config/shield` and enter

        keytool -genkeypair -alias es-ssl -keyalg RSA -keysize 2048 -keypass 
        liferay -storepass liferay -validity 9999 -keystore es-ssl.keystore.jks 
        -ext SAN=DNS:localhost,IP:x.x.x.x,IP:127.0.0.1 -dname "CN=localhost, 
        OU=Organizational Unit, O=Organization, L=Location, ST=State, C=Country"

    Note: Replace the host and IP address with the correct values for your
    system.

3. Add the following lines to `[Elasticsearch_Home]/config/elasticsearch.yml`:

        shield.ssl.keystore.path: /path/to/es-ssl.keystore.jks
        shield.ssl.keystore.password: liferay
        shield.ssl.keystore.key_password: liferay
        shield.transport.ssl: true
        shield.http.ssl: true

    Here you're configuring Shield's SSL properties, including pointing to the
    keystore file you just generated. For more information on these settings,
    read [here](https://www.elastic.co/guide/en/shield/2.2/ssl-tls.html).

4. Update the Shield adapter configuration file you created earlier in
   `Liferay_Home/osgi/configs` by adding these lines:

        requiresSSL=true
        sslKeystorePath=/path/to/es-ssl.keystore.jks

    Now, in addition to enabling authentication, you're enabling SSL encryption
    and pointing Liferay at the keystore file you created for Shield.
    
    Alternatively, you can configure these settings in System Settings. This
    will be mostly useful during development and testing.

5. Start Elasticsearch and Liferay.

Now Shield is fully configured, with both authentication and encryption
protecting your Elasticsearch cluster. Next, you can learn how to [install and configure Marvel](discover/deployment/-/knowledge_base/7-0/marvel),
Elasticsearch's
monitoring plugin, to visualize the health and performance of your Elasticsearch
cluster.

<a name="footnote1">1</a> This is, of course, a nod to all those fans of [Boaty Mcboatface]
