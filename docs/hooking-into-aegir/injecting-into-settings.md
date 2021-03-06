Injecting into settings.php
===========================

Every web site in an Aegir environment has a Drupal configuration file settings.php in /sites/example.com directory. Web administrators often need to make changes to this file; however, the Aegir system also manages this file and any manual customizations will be lost when a site is verified.

Fortunately, there are two mechanisms to ensure that your customizations can be preserved. If you look in the bottom of an Aegir settings.php file you will see references to two files local.settings.php and global.inc.

    <?php
      # Additional host wide configuration settings. Useful for safely specifying configuration settings.
      if (file_exists('/var/aegir/config/includes/global.inc')) {
        include_once('/var/aegir/config/includes/global.inc');
      }

     # Additional site configuration settings. Allows to override global settings.
     if (file_exists('/var/aegir/example-platform/sites/example.com/local.settings.php')) {
       include_once('/var/aegir/example-platform/sites/example.com/local.settings.php');
     }

If these files exist they are loaded at run time by Drupal. As you can probably surmise from the paths to these files, local.settings.php is for site-specific customizations and global.inc is for Aegir-wide customization.

Let's look at these files in more detail. We'll use customization of user session cookies as an example. If you look at the settings.php file generated by Aegir you see that it sets more conservative php settings for cookies (@ini_set('session.cookie_lifetime',  0); i.e. cookies expire immediately) than are in the default.settings.php packaged with Drupal (@ini_set('session.cookie_lifetime',  2000000); i.e. 2 million seconds, which is just over 23 days).
Site-specific Customization

### Site Specific Customization

The local.settings.php file by default does not exist in a new Aegir site installation so you have to create it. Continuing with our example of user session cookies, let's override the Aegir default.

    <?php
      # site-specific Drupal customization

      # override Aegir-generated cookie policy for sites - set cookies to expire after a week (604,800 seconds)
      @ini_set('session.cookie_lifetime', 604800);

Note that because local.settings.php is included after the variables are set in the main settings.php it's customizations takes precedence.

Now, whenever you clone a site or migrate it between platforms, Aegir moves a copy of local.settings.php as well.
Using drush_hook_provision_drupal_config

You can also use the API to add module-specific site configurations with hook_provision_drupal_config

    <?php
      * Append PHP code to Drupal's settings.php file.
      *
      * To use templating, return an include statement for the template.
      *
      * @param $uri
      *   URI for the site.
      * @param $data
      *   Associative array of data from provisionConfig_drupal_settings::data.
      *
      * @return
      *   Lines to add to the site's settings.php file.
      *
      * @see provisionConfig_drupal_settings
      */
      function drush_hook_provision_drupal_config($uri, $data, $config) {
        return '$conf[\'reverse_proxy\'] = TRUE;';
      }

For example it could look like this

    <?php
      function drupalwiki_provision_drupal_config($uri, $data, $config) {
        $extra = drush_get_option('site_extra_settings', '');
      // remove window CR
      $extra = str_replace("\r",'',$extra);
        return $extra;
      }

That is used to add the site settings added by the UI in the hosting backend implemented in [https://github.com/EugenMayer/hosting_site_settings](https://github.com/EugenMayer/hosting_site_settings)

### Aegir-wide Customization

In some situations you may want to implement the same configuration settings on all your Aegir sites. This is where global.inc comes in. Note that global.inc is now included in settings.php before local.settings.php, so that Aegir system administrators no longer retain the ability to override configuration changes in local.settings.php, but instead it is possible to override global settings per site [read why this has been changed](http://drupal.org/node/1044938): this change is available since 0.4-rc1 release.

For example, say the system administrator wanted to limit users' session lifetimes to a maximum of one day they could create a global.inc as follows:

    <?php
    # Aegir-wide Drupal customization

    # override Aegir-generated cookie policy for all sites - set cookies to expire after a day (86,400 seconds)
      @ini_set('session.cookie_lifetime', 86400);

You can even set more granular policy within global.inc (however it makes more sense to keep site-specific overrides in the local.settings.php):

    <?php
    # Aegir-wide Drupal customization

    # override Aegir-generated cookie policy for all sites - set cookies to expire after a day (86,400 seconds)
      @ini_set('session.cookie_lifetime', 86400);

    # Make the aegir front-end server more secure by expiring cookies immediately
      if (preg_match("/hostmaster/", $conf['install_profile'])) {
    # set cookies to expire immediately on hostmaster
        @ini_set('session.cookie_lifetime', 0);
      }


If you are using Aegir to manage multiple remote webservers, you will need to run the Verify task on the webserver in order to push global.inc to the remote machine.