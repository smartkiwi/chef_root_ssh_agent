# <a name="title"></a> root-ssh-agent

## <a name="description"></a> Description

A chef recipe that will allow "sudo su root" to maintain the ssh agent.
This is necessary for vagrant & chef-solo to work with ssh-agent forwarding.

## <a name="usage"></a> Usage

Simply include `recipe[root-ssh-agent]` in your run\_list, and subsequently chef-solo 
running as root (default with vagrant) will have access to your SSH_AUTH_SOCK variable, 
and consequently will have access your running ssh-agent instance.

Please note if a cookbook executes commands as a non-root user (eg chef-homesick), they will not
have permission to access file referenced in `$SSH_AUTH_SOCK`, and forwarding will fail. To work 
around this, we modified the recipe to use `ssh otheruser@localhost` instead of `su otheruser`
as the mechanism to run the command as otheruser. Specifically:  

```ruby
  ssh_options = "-A -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
  cmd_prefix = 'PATH=$PATH:/opt/vagrant_ruby/bin'
  remote_command = "ssh #{ssh_options} #{new_resource.user}@localhost '#{cmd_prefix} #{command}'"
  env = {
    "SSH_AUTH_SOCK" => ENV['SSH_AUTH_SOCK'],
  }
  execute remote_command do
    user "root"
    environment env
  end
```

Of course, this only works if your private key allows you to log-in as that user.


## <a name="requirements"></a> Requirements

### <a name="requirements-chef"></a> Chef

Tested on vagrant and chef-solo.

## <a name="installation"></a> Installation

Nothing specific.

## <a name="recipes"></a> Recipes

### <a name="recipes-default"></a> default

This recipe is a no-op and does nothing.

### <a name="recipes-env-keep"></a> env-keep

Adds the following to `/etc/sudoers.d/root_ssh_agent`: 
    Defaults env_keep += "SSH_AUTH_SOCK" 

### <a name="recipes-ppid"></a> ppid

Uses the `ppid` (parent process id) to find the `$SSH_AUTH_SOCK` path associated with
the parent process (which presumably has the forwarded keys), and sets that as `$SSH_AUTH_SOCK`.

## <a name="notes"></a> Misc notes

See the following resources:
* http://stackoverflow.com/questions/7211287/use-ssh-keys-with-passphrase-on-a-vagrantchef-setup
* http://serverfault.com/questions/107187/sudo-su-username-while-keeping-ssh-key-forwarding#answer-118932

SSH forwarding breaks when the vagrant user runs `chef-solo` as root, unless the following is in place:
    `echo 'Defaults env_keep+="SSH_AUTH_SOCK"' > /etc/sudoers.d/env_keep_sshauth`

Vagrant boxes are supposed to include the following in sudoers, in practice they don't, see https://github.com/mitchellh/vagrant/issues/1151

Debugging tips: 
* `sudo su -` resets all env variables, no matter what /etc/sudoers env_keep specifies. "sudo su" or "sudo su root" doesn't.
* `fail @variable.to_yaml` is a good way to debug a recipe from ruby
* `ssh-add -l && false` is a good way to debug a recipe's command resource
* `sudo VISUAL=vim visudo -f /etc/sudoers.d/env_keep_sshauth


