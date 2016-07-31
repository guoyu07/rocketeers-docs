# Creating tasks

Now that we've informed Rocketeer on how our application and server are structured, it's time to tell it what to do to deploy it. Now, by default it's pretty smart and will know how to install your dependencies and stuff, but that's not all is it?

## Overriding subtasks: database operations

Most common task you can create: database stuff. When Deploy fires, within it it fires _another task_ (tasks can be nested at will like that). That task is called `Migrate`... but by default it does nothing (since migrations are usually framework-specific), so just like we did for `Primer`, we can override it in `hooks.php`.

Imagine we're running a Laravel application, we would need to call `php artisan migrate --force`, and maybe we have a backup command to call beforehand, so let's write our task as such:

**.rocketeer/config/hooks.php**

```php
<?php
'tasks' => [
    'Migrate' => function () {
        $this->runForApplication([
            'php artisan db:backup',
            'php artisan migrate --force',
        ]);
    },
],
```

As I mentioned before, by default `run` runs command in the configured `root_directory`. In local that's the current directory so that's fine but on the server the root directory is the root of the Rocketeer folder (eg. `/home/www/website`), but that's not where we want to run commands, we want to run them in whatever the current release is (eg. `/home/www/websites/releases/123456789`), so for this we use `runForApplication`.

As you can see here we are not returning the results of `run...`, because then Rocketeer would use whatever those commands output to judge whether the task succeeded, but our migration command could have output and still have failed, which would be dangerous.

This is a **very important** concept in Rocketeer: if you return something, whatever it is (array, string, boolean), if it evaluates to truethy the task will be considered a success. So in this case we don't return anything, and in that case Rocketeer will use the exit status of the last command (`artisan db:backup && artisan migrate`) instead.

This means if any of those two task failed and returned an erroneous exit status, our Migrate task would be considered a failure, which is perfect (I mean kinda)!

## Your first real task: clearing caches and compiling stuff

Ok that was relatively easy, we just overwrote a core task like we did for Primer and filled-in whatever logic we needed. But most of the time we need to do more than migrate things for a deployment, where do we do the rest?

This is where events come in. Rocketeer leverages **events** for its tasks, built on top of [league/event](http://event.thephpleague.com). What that means is that, whenever a task fires, it fires a `before` event, and an `after` event we can hook into. Some fire more than that but those two are at least _guaranteed_ to always be there.

If you go back to `hooks.php` you may have noticed we actually have an `events` array pre-created for us, let's see how to use it. We want to per example, clear Laravel's cache after a deployment, so we want to hook `after.deploy` and run `php artisan cache:clear`. Which means our task is going to go here:

**.rocketeer/config/hooks.php**

```php
<?php
return [
  'hooks' => [

    'events' => [
      'after' => [
        'deploy' => [
          // Here!
        ],
      ],
    ],

  ],
];
```

Now as I mentioned in the previous section, a task can be three things: a string/array of strings, a closure, or a class. Let's try to write our task in the three styles to give you an idea.

### Writing a string task

This is the easiest of them all, cause it's just a string:

```php
'after' => [
  'deploy' => [
    'php artisan cache:clear',
  ],
],
```

Done, that was easy; an array of tasks works exactly the same way:

```php
'after' => [
  'deploy' => [
    ['php artisan cache:clear'],
  ],
],
```

### Writing a closure task

Same as before with Primer and Migrate, we write a Closure and use one of the `run` methods (in this case `runForApplication`):

```php
'after' => [
  'deploy' => [
    function () {
      $this->runForApplication('php artisan cache:clear');
    }
  ],
],
```

Notice how we don't return the output of the command here either, we want to trust the exit status instead. Note that you can do this manually too:

```php
'after' => [
  'deploy' => [
    function () {
      $this->runForApplication('php artisan cache:clear');

      return $this->status();
    }
  ],
],
```

The `status` method returns a boolean, `true` if the exit status was 0, `false` otherwise. But what if you want to do something when it fails? Or display a custom error message? Well in those cases, you can use the `halt($message)` method, which will manually fail the task and display a more context-specific error message:

```php
'after' => [
  'deploy' => [
    function () {
      $this->runForApplication('php artisan cache:clear');

      return $this->status() || $this->halt('Failed to clear the cache');
    }
  ],
],
```

### Writing a class task

Now let's dive into the belly of the beast, let's write a task class. If you look into your `.rocketeer` folder, you should see a `tasks.php` file created in the `app` folder. This is a folder whose contents are automatically loaded on boot, no matter which files, no matter what's in them. Rocketeer creates two files during ignition (tasks and events) but you can name them however you want, create subfolders, etc.

In order to write a task class, you need to write a class that extends `Rocketeer\Tasks\AbstractTask`. The core logic of the task will go into the `execute` method – your IDE should already suggest for you to implement it. So let's do so:

**.rocketeer/tasks.php**

```php
<?php
class ClearCache extends Rocketeer\Tasks\AbstractTask
{
    public function execute()
    {
        $this->runForApplication('php artisan cache:clear');

        return $this->status() || $this->halt('Failed to clear the cache');
    }
}
```

Now, to register that task, we can just use its qualified name in our `hooks.php` file like this:

```php
'after' => [
  'deploy' => [
    ClearCache::class
  ],
],
```

And there we go, if we try to deploy now, as soon as our deploy is finished, our cache will be cleared.

## A note about deploy events

So far I've been saying to hook your tasks into `after.deploy` to ease you into the idea of events but that's not actually entirely correct.

When you deploy, as we've seen, the **Deploy** tasks will fire a bunch of other tasks, the last one being **SwapSymlink** which makes the `current/` folder symlink to the new release we just created. This means if you hook after the deploy, the symlink will already be changed and your website will already be online by the time you execute your commands on it.

What you actually want to do, is execute your commands _before_ the symlink is swapped. So since our task is **SwapSymlink**, the event would be `before.swap-symlink`:

```php
'before' => [
  'swap-symlink' => [
    ClearCache::class
  ],
],
```