
satellite_template_synchronization
=========

This role configures [Satellite's Template Synchronization Plugin](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.14/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#doc-wrapper).
It performs the steps outlined in the [documentation](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.14/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#synchronizing-templates-with-an-existing-repository_managing-hosts).

It makes use of the Red Hat certified collection [`redhat.satellite`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/docs/).

This role will *not* run the `satellite-installer` to enable the plugin. This can be done using the 
[`redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/) role.

To use the certified collection `redhat.satellite` you need to be a Red Hat subscriber. If you don't own any subscriptions, you can make use of
[Red Hat's Developer Subscription](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux) which is provided at no cost by Red Hat.

You can also make use of the upstream collection [`theforeman.foreman`](https://docs.ansible.com/ansible/latest/collections/theforeman/foreman/index.html), but you'd need to change the module names from `redhat.satellite` to `theforeman.foreman` - but I
have *not* tested this.

Additionally, you can optionally deploy the generated SSH key to your GitHub repository where you host the templates. For an example repository containing Partition Tables and Provisioning Templates, please take a look at my repository
[satellite_templates](https://github.com/sscheib/satellite_templates).

This role is designed in such a way that adding future source code management (SCM) platforms (e.g. GitLab, BitBucket) is easy and straight forward. Please look at `tasks/github.yml` and `tasks/assert_github.yml` to understand how it works.

The structure how to pass the settings for the Template Synchronization Plugin seems at first counter-intuitive, but the role is designed in such a way that you can pass it the same settings definition you might have passed to the
[redhat.satellite.settings](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/settings/) role. This role uses a different variable name, however: `sat_template_sync_settings`.

Passing it the definition of the role `redhat.satellite.settings` is as simple as: `sat_template_sync_settings: '{{ satellite_settings }}'`.
Only specific settings are extracted from that list of dictionaries, so you don't need to redefine anything. Please take a look at the
example portion of this file to see what a this list of dictionaries can look like.

Requirements
------------

All collection requirements are provided via `collections/requirements.yml`, which are:
1. `redhat.satellite`: Interacting with the Satellite to configure the Template Sync Plugin
2. `community.crypto`: Generate the SSH key for the foreman user (optional)
3. `community.general`: Create a deployment SSH key for the given templates repository on GitHub (optional)

To synchronize your templates, it is required to have the `ERB`-Templates for Satellite stored in a git repository. For an example repository containing Partition Tables and
Provisioning Templates, please take a look at my repository [satellite_templates](https://github.com/sscheib/satellite_templates).

Lastly, your Red Hat Satellite needs to be installed with the
[Template Sync Plugin](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.14/html/managing_hosts/synchronizing_templates_repositories_managing-hosts) enabled.
This can be done using the 
[`redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/) role.

Role Variables
--------------
| variable                                     | default                      | required | description                                                                    |
| :---------------------------------           | :--------------------------- | :------- | :----------------------------------------------------------------------------- |
| `sat_template_sync_settings`                 | unset                        | true     | Settings to use when synchronizing templates. Check for an example below       |
| `satellite_username`                         | unset                        | true[^1] | Username to authenticate with against the Satellite API                        |
| `satellite_password`                         | unset                        | true[^1] | Password of the user to authenticate with against the Satellite API            |
| `satellite_server_url`                       | unset                        | true[^1] | URL to the Satellite API (including http/s://)                                 |
| `satellite_validate_certs`                   | `true`                       | false    | Whether to validate certificates when connecting to the Satellite API          |
| `sat_foreman_username`                       | `foreman`                    | false    | Foreman username (usually `foreman`)                                           |
| `sat_create_foreman_ssh_key`                 | `false`                      | false    | Whether to create an SSH key for the foreman user                              |
| `sat_foreman_ssh_key_path`                   | check `defaults/main.yml`    | false    | Path to the foreman user's private SSH key                                     |
| `sat_foreman_ssh_public_key_path`            | check `defaults/main.yml`    | false    | Path to the foreman user's public SSH key (used for deploying to an SCM)       |
| `sat_foreman_ssh_key_type`                   | `rsa`                        | false    | SSH key type to generate. Currently only `rsa` supported                       |
| `sat_foreman_ssh_key_size`                   | `4096`                       | false    | Size/length of the SSH key to generate                                         |
| `sat_add_repo_host_keys_to_known_hosts`      | `false`                      | false    | Whether to add scanned keys from the repo URL to the `known_hosts` of foreman  |
| `sat_foreman_known_hosts_file_path`          | check `defaults/main.yml`    | false    | Path to the `known_hosts` file of the foreman user                             |
| `sat_deploy_foreman_ssh_key`                 | `false`                      | false    | Deploy the SSH public key to an SCM (only GitHub supported at the moment)      |
| `sat_quiet_assert`                           | `true`                       | false    | Whether to quiet assert                                                        |
| `sat_template_sync_lock_allowed_values`      | check `defaults/main.yml`    | false    | Allowed values for attribute `template_sync_lock` when defined as string       |
| `sat_template_sync_associate_allowed_values` | check `defaults/main.yml`    | false    | Allowed values for attribute `template_sync_associate` when                    |
| `sat_template_sync_organizations`            | unset                        | false    | List of Organizations to add the imported Templates to                         |
| `sat_template_sync_locations`                | unset                        | false    | List of Locations to add the imported Templates to                             |
| `sat_scm_type`                               | unset                        | false    | Which SCM to use to deploy the foreman SSH public key (only `github` right now)|
| `sat_github`                                 | unset                        | false    | GitHub variables for deploying the SSH key to GitHub (check below example)     |

[^1]: This role also supports passing the Satellite URL, username and password via environment variables. Please check `vars/main.yml` for the exact variables to use.

## Variable `sat_template_sync_settings`

An extended example of only the `sat_template_sync_settings` variable is illustrated down below:
```
sat_template_sync_settings:
  - name: 'template_sync_verbose'  # whether to be verbose when synchronizing Templates
    value: true

  - name: 'template_sync_associate' # whether to associate Templates with OSes, Locations and Organizations on import
    value: 'always'

  - name: 'template_sync_filter'  # only import Templates that match the filter. Supports regular expressions.
    value: '^(pv?t|snt)-'

  - name: 'template_sync_repo'  # git repository to synchronize the Templates from
    value: 'ssh://git@github.com/sscheib/satellite_templates.git'

  - name: 'template_sync_negate'  # whether to negate the filter
    value: false

  - name: 'template_sync_branch'  # branch to select when synchronizing
    value: 'main'

  - name: 'template_sync_force'  # whether to force the import, which overrides every changes performed on the Templates in the Satellite directly
    value: true

  - name: 'template_sync_lock'  # whether to lock the imported Templates
    value: 'lock'

  - name: 'template_sync_dirname'
    value: '/'

  - name: 'template_sync_prefix'
    value: 'custom_prefix'
```

All Template settings above can be mixed and matched. The `name` attribute of each setting is the same as can be passed to the role `redhat.satellite.settings`.

A validation is done on each possible settings you can define for the Template Sync plugin (which are the ones above). Some settings only support specific string values, which
are:
- `template_sync_associate`: Allowed values are: `always`, `new`, `never`
- `template_sync_lock`: Allowed values are: `lock`, `keep_lock_new`, `keep`, `unlock`

The setting `template_sync_lock` can be defined as boolean, which then needs to be either `true` or `false` (specified in your preferred variant [`yes`, `1`, etc.]).

The allowed values for both `template_sync_associate` and `template_sync_lock` can be overriden with `sat_template_sync_lock_allowed_values` and
`sat_template_sync_associate_allowed_values` should the API definition of the Satellite change in the future.

Please note that associating Templates with OSes, Locations and Organizations requires a specific [Template header](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.13/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#Importing_Templates_managing-hosts) for each Template.

## Variable `sat_github`

An extended example of only the `sat_github` variable is illustrated down below:
```
sat_github:
  owner: !vault |
         $ANSIBLE_VAULT;1.1;AES256
         [..] 
  repository: 'satellite_templates'
  key_name: 'foreman-{{ inventory_hostname }}'
  read_only: true
  force_deployment: false
  token: !vault |
         $ANSIBLE_VAULT;1.1;AES256
         [..]
```

Dependencies
------------

None

Example Playbook
----------------

```
---
- name: 'Synchronize Satellite Templates using the Template Sync plugin'
  hosts: 'all'
  gather_facts: false
  roles:
    - role: 'satellite_template_synchronization'
      vars:
        satellite_server_url: 'https://satellite.example.com'
        satellite_username: 'steffen'
        satellite_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [..]
        satellite_validate_certs: true
        sat_foreman_username: 'foreman'
        sat_create_foreman_ssh_key: true
        sat_foreman_ssh_key_path: '/usr/share/foreman/.ssh/id_rsa'
        sat_foreman_ssh_public_key_path: '/usr/share/foreman/.ssh/id_rsa.pub'
        sat_foreman_ssh_key_type: 'rsa'
        sat_foreman_ssh_key_size: 4096
        sat_add_repo_host_keys_to_known_hosts: true
        sat_foreman_known_hosts_file_path: '/usr/share/foreman/.ssh/known_hosts'
        sat_deploy_foreman_ssh_key: true
        sat_template_sync_organizations
          - 'org-example_org'
          - 'org-example_org2'

        sat_template_sync_locations
          - 'loc-example_loc1'
          - 'loc-example_loc2'

        sat_template_sync_lock_allowed_values:
          - 'lock'
          - 'keep_lock_new'
          - 'keep'
          - 'unlock'

        sat_template_sync_associate_allowed_values:
          - 'always'
          - 'never'
          - 'new'

        sat_template_sync_settings:
          - name: 'template_sync_verbose'
            value: true

          - name: 'template_sync_associate'
            value: 'always'

          - name: 'template_sync_filter'
            value: '^(pv?t|snt)-'

          - name: 'template_sync_repo'
            value: 'ssh://git@github.com/sscheib/satellite_templates.git'

          - name: 'template_sync_negate'
            value: false

          - name: 'template_sync_branch'
            value: 'main'

          - name: 'template_sync_force'
            value: true

          - name: 'template_sync_lock'
            value: 'lock'

          - name: 'template_sync_dirname'
            value: '/'

          - name: 'template_sync_prefix'
            value: 'custom_prefix'

        sat_scm_type: 'github'
        sat_github:
          owner: 'sscheib'
          repository: 'satellite_templates'
          key_name: 'foreman-{{ inventory_hostname }}'
          read_only: true
          force_deployment: false
          token: !vault |
                  $ANSIBLE_VAULT;1.1;AES256
                  [..]
        sat_quiet_assert: true
...
```

License
-------

GPL-2.0-or-later
