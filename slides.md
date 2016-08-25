<!-- $theme: gaia -->
<!-- page_number: false -->

# No Hugging, No Learning

Olaf Alders

Toronto Perl Mongers

Aug 25, 2016

@olafalders (Twitter)
oalders (GitHub)
OALDERS (PAUSE)

https://github.com/oalders/no-hugging-no-learning

---

# The Problem

* Building an app that can track and chart (almost) anything that is available via 3rd party APIs
  * No screen scraping
  * No ToS violations
* Doing this in my limited spare time

---

# The Solution

* Build an app based solely on what I already know: "No hugging, no learning"
* Of the three programmer virtues (laziness, impatience and hubris), I would let laziness be my guiding light
* I tried to self-impose a real lack of intellectual curiosity

---

# The Evolution

---

## The App

### Before
* Some command line scripts as a proof of concept
* This became a Dancer (1) app run via `Plack`

---

#### Pros
* Dancer apps are easy to get up and running

#### Cons
* I didn't really love the `Dancer` (1) syntax
* `Dancer2` was still quite new and the plugin support wasn't there yet

---

### After
* I moved to a `Mojolicous Lite` app run via `Plack`
* I then transitioned to a full `Mojo` app
  * Run via `morbo` in development
  * Run via `hypnotoad` in production

---

### morbo
* By default watches your filesystem for changes and then reloads your app


---

### hypnotoad
* Zero downtime restarts
* You don't need to provide your own init script
* If you don't care about zero downtime or just want the prefork server, you have that option
  * `perl script/myapp prefork`
    * same config args as morbo/daemon

---

### Did I learn?

* I partly became familiar with `Mojo` via my day job, but I did have to do a fair bit of learning on my own. 
* I was not familiar with `Hypnotoad`, but luckily that was a pretty easy transition.  The docs are really good.

---

## Authentication

---

### Before
* Mozilla's `Persona`

---

#### Pros
* Pretty easy to configure
* All anyone really needs is an email address

#### Cons
* Issues getting it to work offline
* No widespread adoption
* Felt like the first 80% was easy but the last 20% was eating into my time

---

### After
* OAuth via any of the available integrations
* I needed this anyway in order to get user data, so it keeps things simpler to have `Persona` out of the mix
* With Mozilla no longer 100% behind Persona, it becomes a less attractive solution 
  * most non-technical users won't be familiar with it at this point anyway

---

## SSL Everywhere

* Looked briefly into Let's Encrypt via Ansible
* Ended up spending USD 10 for a RapidSSL cert

---

## Relational Database

---

### Before
* Application data stored in `MySQL` database

---

#### Pros
* I'm very familiar with `MySQL`
* I prefer many `MySQL` tools, like the `mysql` command line interface and `phpMyAdmin`
* `MySQL` replication is dead easy to configure

#### Cons
* "Foreign keys that point to the renamed table are not automatically updated. In such cases, you must drop and re-create the foreign keys in order for them to function properly."

---

### After

* 3 `Postgres` databases
  * `Minion`
  * Application data
  * Test suite fixtures

---

### Did I learn?

* Since we use `Postgres` at `$work`, I didn't have to learn a whole lot to make the switch.
* The database does a litle bit more than the bare minimum -- I'm not taking advantage of all that Postgres has to offer
  * No `jsonb` columns just yet 

---


## Job Management

---

### Before
* cron scripts

---

### After
* `Minion`

---

### Did I learn?

* Had basically zero knowledge of Minion implementation  
* But, there's not much you need to learn in order to get up and running
* Minimized a lot of really convoluted cron job logic
* This probably saved me  time in the long run

---

## Minion How To
* You don't need to have a full-blown Mojo app to use Minion
* Just create a bare bones app to get started
  * Use this with your Catalyst, Dancer, [insert framework here] application
* Let's look at how MetaCPAN uses Minion

---

## Create an App
* https://github.com/metacpan/metacpan-api/blob/master/lib/MetaCPAN/Queue.pm
  * Sets up a tiny Mojo app
  * Adds Minion tasks in the `startup()` method
  * You can see how we trap warnings and log failed jobs in the `_gen_index_task_sub()` method

---

## Optionally Create Your Minion Backend Elsewhere
* We've abstracted it out to https://github.com/metacpan/metacpan-api/blob/master/lib/MetaCPAN/Queue/Helper.pm
* We do this so that we can use an `SQLite` database when running under the test harness and a `Postgres` database in all other cases
  * Using `SQLite` for testing makes our `Travis` configuration much easier
  * This will also allow us more easily to run tests in parallel

--- 

## Start Up Your Queue
* In development, you can create a 4-5 line script to start up your queue via `morbo`
   * https://github.com/metacpan/metacpan-api/blob/master/bin/queue.pl
* In production we use `Daemon::Control` to manage starting and stopping a daemon 
  * https://github.com/metacpan/metacpan-puppet/blob/master/modules/minion_queue/templates/init.pl.erb

---

## Add Tasks

* A task is essentially an anonymous sub which you can later refer to by name
* A job is an instance of a task

```perl
$minion->add_task( add_things => sub {

  # $job is a Minion::Job object
  my ($job, $first, $second) = @_;
  $job->finish($first + $second);
});

```

---

## More Complete Example
```perl
$minion->add_task( add_things => sub {
    my ($job, $first, $second) = @_;
    $job->finish({
        message => 'Great!',
        total   => $first + $second, 
    });
});

my $id = $minion->enqueue(add_things => [1, 1]);

my $result = $minion->job($id)->info->{result};
```

`$result` is now

```perl
{  message => 'Great!', total => 2 }
```

---

## Storing Your Job Result
* `$job->finish;`
* `$job->finish('Fantastic!');`
* `$job->finish({ foo => 'bar' });`

Later, you can find this arbitrary data in `$job->result`

---

## Populate Your Queue

* enqueue() means "add job to queue"
```
$app->minion->enqueue( my_task_name => ['foo', 'bar'] );
```

```perl
$self->minion->enqueue(
  track_service_resource =>
    [{ service_resource_id => $service_resource->id }] =>
    { priority => 1 },
);
```

* priority ranges from 0-X, where X is a positive integer
* jobs with priority 10 will run before jobs with priority 9 etc

---

## enqueue() options
* `attempts => 10` (# of times to attempt job)
* `delay => 3600` (delay this job for 3600 seconds)
* `parents => [$id1, $id2]` (One or more existing jobs this job depends on, and that need to have transitioned to the state `finished` before it can be processed)
* `priority => 3` (see previous slide)
* `queue => 'dexter'` (some arbitrary name, defaults to `default`)

---

## Minion::Job

https://metacpan.org/pod/Minion::Job

* You can set up events to be fired on `failed` and `finished` states
* Get information via `$job->info`
* `$job->is_finished`
* `$job->remove`
* Much, much more

---

## Inspect Your Queue

On the MetaCPAN Vagrant box:

```sh
$ vagrant ssh
$ cd /home/vagrant/metacpan-api/
$ ./bin/run bin/queue.pl minion job -s
{
  "active_jobs" => 0,
  "active_workers" => 0,
  "delayed_jobs" => 0,
  "failed_jobs" => 0,
  "finished_jobs" => 0,
  "inactive_jobs" => 0,
  "inactive_workers" => 1
}
```

---

## Test Your Queue

https://github.com/metacpan/metacpan-api/blob/master/t/queue.t

```perl
use MetaCPAN::Queue;
use Test::More;
use Test::RequiresInternet ( 'cpan.metacpan.org' => 443 );

my $app = MetaCPAN::Queue->new;

my $release = 'https://cpan.metacpan.org/authors/id/O/OA/OALDERS/HTML-Restrict-2.2.2.tar.gz';

$app->minion->enqueue( 
    index_release => [ '--latest', $release ] 
);

$app->minion->perform_jobs;
done_testing();
```

---

## Advanced: Behaviour on Events

https://github.com/jberger/Minion-Notifier/blob/master/lib/Minion/Notifier.pm#L39-L56

---

```perl
sub setup_worker {
  my $self = shift;

  my $dequeue = sub {
    my ($worker, $job) = @_;
    my $id = $job->id;
    $self->transport->send($id, 'dequeue');
    $job->on(finished => sub { $self->transport->send($id, 'finished') });
    $job->on(failed   => sub { $self->transport->send($id, 'failed') });
  };

  $self->minion->on(worker => sub {
    my ($minion, $worker) = @_;
    $worker->on(dequeue => $dequeue);
  });

  return $self
}
```

---

## Code Context 

The preceding code is called like this:

https://github.com/jberger/Minion-Notifier/blob/master/lib/Mojolicious/Plugin/Minion/Notifier.pm#L37

---

## Pros
* We can replace a failure-prone, forking, long running process with a series of jobs
* If a job fails in an unexpected way, it doesn't prevent all the other jobs from running
* We can check the status of all jobs at any point to gauge how things are progressing
* Jobs which fail can be retried at intervals
* We can start an arbitrary number of worker apps
* Using `Postgres` replication MetaCPAN can now start X workers on 3 different boxes, which gives us greater scaling when needed

---

## Cons
* Minion doesn't come with a handy daemon handler like `hypnotoad`, but you can set this stuff all up yourself pretty easily
* There aren't yet a lot of tools to visualize what's going on with the queue, but again this can be done pretty easily if you need it.

---

## Database Migrations

---

### Before
* Database schema managed by `Database::Migrator`

---

#### Pros
* `Database::Migrator` is what we use at `$work`
* It's pretty simple
* In addition to SQL, you can also run Perl scripts as part of a migration

#### Cons
* `Database::Migrator` *needs to be subclassed*, which means you have to provide a fair bit of functionality just to get up and running

---

### After
* Application and fixture databases managed via `App::Sqitch`
* `Minion` has its own schema deployment logic

---

#### Pros
* `sqitch` handles rollbacks as well as deployment
* Optionally can use custom assertions to ensure that deployments were successful

#### Cons
* There's a lot more going on here, so you probably have to do a bit of reading before just jumping in
* I had to use the CLI rather than the modules because of some weirdness between `Moo` and `Moose` when I tried to use the modules directly

---

### Did I learn?

* Yes, I did have to learn `sqitch` from scratch
* The author (David Wheeler) was extremely helpful
* I was able to get it working with my fixtures

---

### See Also
* Mojo::Pg::Migrations
* Mojo::mysql::Migrations
* Mojo::SQLite::Migrations

---

## Automation of Deployment

---

### Before
* The plan was to use `Puppet`

---

#### Pros
* I was already familiar with `Puppet`

#### Cons
* I was already familiar with `Puppet`
* Even simple things seemed hard
  * Tired of dealing with hard to debug certificate issues
  * There may have been obvious and easy fixes to my problems, but I couldn't easily find them
*  I'm not saying that `Puppet` isn't a wonderful tool, but it felt like a bigger hammer than I needed

---

#### After
* Ansible

---

#### Pros
* Excellent integration with `Vagrant`
* No need for bootstrapping on the target deployment machines
* It's easily installed via `Homebrew` on OS X
* Has good handling of `git submodule`
* User contributed plugins are trivial to install

---

#### Cons
* I actually don't have a lot of complaints
* Moving from v1 to v2 addressed the few issues that I ran into

---

### Ansible in Vagrantfile

```ruby
  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    ansible.playbook = "ansible/site.yml"
    ansible.sudo = true
    ansible.groups = {
        "development" => ["default"],
        "webservers"  => ["default"],
    }
  end
```

---

### Installing Packages

```yaml
---
- gather_facts: no
  hosts: webservers
  tasks:
    - apt: 'pkg={{item}} state=installed'
      name: Install misc apt packages
      with_items:
        - curl
        - ntp
        - tree
        - unzip
    - apt: update_cache=yes cache_valid_time=3600
      name: Run the equivalent of "apt-get update"
    - apt: upgrade=dist autoremove=true
      name: Update all packages to the latest version
```

---

### Did I learn?
Yes, I had to learn all about `Ansible`, since I had never used it before.  In the meantime `$work` has also switched from `puppet` to `ansible`, so it turned out to be knowledge that I can apply there as well.

Keeping in mind how many hours I've spent battling `Puppet` on past projects, I think learning about `Ansible` was actually the better, faster choice.

---

## Predictions Which Did Not Change

---

### JS-Driven Charts
* `Highcharts` turned out to be an excellent choice
  * Excellent docs
  * Responsive support
  * I'm also quite happy with the product itself

---

### Twitter Bootstrap
* Still a solid choice for a CSS framework

---

### Travis CI
* Private repository support is not within my budget (starts at USD 129/month)
* Open Source option works excellently for https://github.com/oalders/wundercharts-plugin

---

### Carton for Dependency Pinning
* This suits my needs fine at this point
* https://metacpan.org/pod/Carmel is still listed as experimental

---

### Vagrant
* Initially I just ran the app inside OS X directly
* I realized pretty quickly that I wanted to have a sandbox
* I've even got a `Vagrantfile` for the plugin system https://github.com/oalders/wundercharts-plugin/blob/master/Vagrantfile
  * So, anyone who wants to contribute can have a fully operational, sandboxed system with all of the modules installed within a matter of minutes

---

### Code::TidyAll
* I use this to tidy as much code as possible
* It's enforced via a Git pre-commit hook so that I don't have to think about it

---

### LWP::ConsoleLogger
* I use this a lot when debugging interactions with 3rd party APIs
* This only really works when you can get at the UA which a module is using
* I wish more module authors would expose their user agents
* The Mojo equivalent is setting `MOJO_USERAGENT_DEBUG=1`

---

### DBIx::Class
* Use foreign keys and the schema dumper automatically sets up relationships for you
* Use this trick to enable `perltidy` to tidy your schema files: https://metacpan.org/pod/release/ILMARI/DBIx-Class-Schema-Loader-0.07045/lib/DBIx/Class/Schema/Loader/Base.pm#filter_generated_code

---

### Mojolicious::Plugin::AssetPack
* Compress and convert css, less, sass, javascript and coffeescript files
* In development mode you get the individual files in your template (for easier debugging)

---

### Strict Constructors
* MooseX::StrictConstructor
* MooX::StrictConstructor
* MetaCPAN::Moose

---

### etc
* `nginx`
* `Ubuntu` 14.04.5 LTS
* `Perl` v5.18.2

---

## Things I Hadn't Planned on Using
---

### Wercker
* I didn't know this existed
* I'm not good about running my test suite after every change, so I realized I had to automate this
* I didn't want to run my own Jenkins/TeamCity/etc service
  * I've seen how much configuration can go into these things and I didn't have the time to mess with it
* Free, concurrent builds for private repositories

---
### Wercker (cont'd)
* I didn't want to give it access to all of my private repositories
  * I created a BitBucket account (free private repositories)
  * Added a new remote to my repository "bitbucket"
  * Every time I push to the "bitbucket" remote the tests are run and I get an email about success or failure
  * Protip: if weird errors happen, sometimes you have to clear your Wercker cache

---

### wrapbootstrap.com
* I hadn't even thought about the UI when I started this project
* I bought the "Inspina" theme for USD 18
  * This gave me solid options for both logged in and logged out UI
  * Was trivial to implement and immediately made my terrible design much better
  * Since I was already using Bootstrap there weren't a lot of changes to be made

---

### Date Range Picker
* http://www.daterangepicker.com
* Easy to use and makes it easy to use sensible defaults when selecting dates or ranges of dates

---

## Open Source That Came Out of This
* Mojolicious::Plugin::Web::Auth::Site::Spotify
* Mojolicious::Plugin::Web::Auth::Site::Runkeeper
* WebService::HealthGraph
* Code::TidyAll::Plugin::YAML
* WWW::Spotify (co-maint)
* Patch to WebService::Instagram
* I open sourced all of my plugins

---

## WunderCharts Plugins
* https://github.com/oalders/wundercharts-plugin
* 100% open source
* I'm in a position to integrate user contributed patches without open sourcing the entire code base

---

* For example, if we want to integrate FitBit
  * Need to create Mojolicious::Plugin::Web::Auth::Site::FitBit
  * Need to send a pull request for WunderCharts::Plugin::FitBit


---
## Third Party Integrations
* I didn't have a firm idea of how many I wanted
* Got a mixture of OAuth, OAuth2 and no auth

---

* OAuth
  * Twitter
* OAuth 2
  * Facebook
  * Instagram
  * Spotify
  * GitHub
* No authentication required
  * Hacker News

---

## Don't Count Your Chickens
* Instagram apps (unlike the other sites) have a formal review process
* I didn't realize until _after_ my integration was finished that my use case may not qualify
* I _did_ get accepted at the end of the day, but it's a bit of a coin toss
* You need to show them the integration via a screen cast before you get approval, so it's possible to do the heavy lifting and still be denied

---

## Success?

Among other things, I ended up having to learn (or learn more about) the following technologies:

* Ansible
* BitBucket
* Date Range Picker
* hypnotoad
* Minion
* Mojo
* Mojolicious::Plugin::AssetPack

---
* Persona
* sqitch
* Wercker
* [also various 3rd party APIs for web services]

---

## Thanks!

* To Joel Berger for proofreading the Mojo bits of an earlier version of this talk
* All remaining (or since introduced) errors are my own
* 
---

## The "Finished" Product

https://wundercharts.com

