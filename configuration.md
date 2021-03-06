# Configuration

To setup configuration param use `set` function, and to get it inside task use `get` function.

~~~php
set('param', 'value');

task('deploy', function () {
    $param = get('param');
});
~~~

Each param can be overridden for each host:
  
~~~php
host(...)
    ->set('param', 'new value');
~~~

Configuration parameters also can be specified as callback function, which will be executed on remote host on first `get` call:

~~~php
set('current_path', function () {
    return run('pwd');
});
~~~

You can use param's values inside `run` calls with `{{ }}`, instead of doing this:

~~~php
run('cd ' . get('release_path') . ' && command');
~~~

You can do it:

~~~php
run('cd {{release_path}} && command');
~~~

Common recipe comes with a few predefined config params listed below. 

To get list of available params run:

~~~sh
dep config:dump
~~~

Show current deployed release:

~~~bash
dep config:current
~~~

Show inventory:

~~~bash
dep config:hosts
~~~



Below is a list of common variables.

### deploy_path

There to deploy application on remote host. You should define this variable for all of your hosts.
For example, if you want to deploy your app to home directory:

~~~php
host(...)
    ->set('deploy_path', '~/project');
~~~

### hostname

Current hostname. Automatically sets by `host` function.

### user

Current user name. Default to current git user name:

~~~php
set('user', function () {
    return runLocally('git config --get user.name');
});
~~~

You can override it in _deploy.php_ for example to use env var:
 
~~~php
set('user', function () {
    return getenv('DEP_USER');
});
~~~

`user` parameter can be used to configure notification systems:

~~~php
set('slack_text', '{{user}} deploying {{branch}} to {{hostname}}');
~~~

### release_path

Full path to current release directory. Current dir path in non-deploy contexts.
Use it as working path for your build:

~~~php
task('build', function () {
    cd('{{release_path}}');
    // ...
});
~~~

> By default, working path is `release_path` for simple task:
> ~~~php
> task('build', 'webpack -p');
> ~~~

### previous_release

Points to previous release if it exists. Otherwise variable doesn't exists.

~~~php
task('npm', function () {
    if (has('previous_release')) {
        run('cp -R {{previous_release}}/node_modules {{release_path}}/node_modules');
    }
    
    run('cd {{release_path}} && npm install');
});
~~~

### ssh_multiplexing

Use [ssh multiplexing](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) for speedup native ssh client.

~~~php
set('ssh_multiplexing', true);
~~~

### default_stage

If hosts declaration have stages, this option allow you to select default stage to deploy with `dep deploy`.

~~~php
set('default_stage', 'prod');

host(...)
    ->stage('prod');
~~~

You can also set callable as argument if you need some more complex ways to determine default stage.

Having callable in set() allows to not set the value at once when declared but later when used. There is no difference 
when we assign simple string but when we assign value of function then this function must be called at once if not used 
as callable. With callable it can be called when used so function which determine variable can be overwritten by user 
with its own function. This is the great power of having callable in set() instead of direct function calls.

**Example 1: Direct function assign in set()**

Lets assume that we must include some third party recipe that is setting 'default_stage' like that:
~~~php
set('default_stage', \ThirdPartyVendor\getDefaultStage());
~~~

And we want to overwrite this in our deploy.php with own value:
~~~php
set('default_stage', \MyVendor\getDefaultStage());
~~~

Third party recipe should avoid direct function call because it will be called always even if overwrite it with 
our own set('default_stage', \MyVendor\getDefaultStage()). Look for next example how third party recipe should use
callable in that case.

**Example 2: Callable assign in set()**

Lets assume that we must include some third party recipe that is setting 'default_stage' like that:
~~~php
set('default_stage', function() {
    return \ThirdPartyVendor\getDefaultStage();
});
~~~

And we want to overwrite this in our deploy.php:
~~~php
set('default_stage', function() {
    return \MyVendor\getDefaultStage();
});
~~~

Result is that only \MyVendor\getDefaultStage() is run.

### keep_releases

Number of releases to keep. `-1` for unlimited releases. Default to `5`.

### repository

Git repository of application.

To use a private repository it needs to generate a SSH-key on your host and add to the repository
as a Deploy Key (a.k.a. Access Key). This key allows your host to pull out the code. Or use can use agent forwarding. 

Note that at the first time host can ask to add host in `known_hosts` file. The easiest way to do it is
running `git clone <repo>` on your host and saying `yes`.

### git_tty

Allocate TTY for `git clone` command. `false` by default. This allow you to enter passphrase for keys or add host to `known_hosts`.

~~~php
set('git_tty', true);
~~~

### git_recursive

Set the `--recursive` flag for git clone. `true` by default. Setting this to false will prevent submodules from being cloned as well.

~~~php
set('git_recursive', false);
~~~

### branch

Branch to deploy.

If you want to deploy a specific tag or a revision, you can use `--tag` and `--revision` options while running `dep deploy`. F.e.

~~~bash
dep deploy --tag="v0.1"
dep deploy --revision="5daefb59edbaa75"
~~~

Note that `tag` has higher priority than `branch` and lower than `revision`.

### shared_dirs

List of shared dirs.

~~~php
set('shared_dirs', [
    'logs',
    'var',
    ...
]);
~~~

### shared_files

List of shared files.

### copy_dirs

List of files to copy between release.

### writable_dirs

List of dirs which must be writable for web server.

### writable_mode

Writable mode

* `acl` (*default*) use `setfacl` for changing ACL of dirs.
* `chmod` use unix `chmod` command,
* `chown` use unix `chown` command,
* `chgrp` use unix `chgrp` command,

### writable_use_sudo

Use or not `sudo` with writable command. Default to `false`.

### writable_chmod_mode

Mode for using then `writable_mode` in `chmod`. Default: `0755`.

### writable_chmod_recursive

Whether to set `chmod` on dirs recursively or not. Default: `true`.

### http_user

User of web server. If this parameter does not configured, deployer try to detect it from process list. 

### clear_paths

List of paths which need to be deleted in release after updating code. 

### clear_use_sudo

Use or not `sudo` with clear_paths. Default to `false`.

### cleanup_use_sudo

Use or not `sudo` with `cleanup` task. Default to `false`.

### use_relative_symlink

Use or not relative symlinks. By default deployer will detect if system supports relative symlinks and use it.

> Relative symlink used by default, if your system supports it.

### use_atomic_symlink

Use or not atomic symlinks. By default deployer will detect if system supports atomic symlinks and use it.

> Atomic symlinking used by default, if your system supports it.

### composer_action

Composer action. Default `install`.

### composer_options

Options for composer.

### env

Array of environment variables.

~~~php
set('env', [
    'VARIABLE' => 'value',
]);
~~~


Read more about [task definitions](tasks.md).
