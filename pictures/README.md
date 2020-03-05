# bento-jobs

Builds are handled by [Jenkins pipeline on ree-bento](https://ree-bento.aws-dev.manheim.com/job/MAN-Tooling/job/MAN-Tooling%20Org/job/bento-jobs/)

This repo contains a collection of Jenkins startup scripts and utility jobs.
All of this originated and has been pulled from the original
[Bento Repository](https://ghe.coxautoinc.com/MAN-Tooling/bento).

``config/bento_local_jobs.groovy`` and ``config/init/common_pipeline_seed.groovy``
are currently fetched and executed during start up of the
[manheim-jenkins-master docker image](https://ghe.coxautoinc.com/MAN-Tooling/manheim-jenkins-master)

## Versioning

To allow us to update this repo without breaking existing images/bentos we have
started to use release tags.  This tag is referenced in our seed job,
[manheim-jenkins-master build/create_bootstrap_job.groovy](https://ghe.coxautoinc.com/MAN-Tooling/manheim-jenkins-master/blob/master/build/create_bootstrap_job.groovy#L8).

If a new release tag is to be created, the branch args in bento_local_jobs.groovy
and bento_central_jobs.groovy should be updated to reflect this tag. If you want
that new version to be _used_, you will need to release a new version of [manheim-jenkins-master](https://ghe.coxautoinc.com/MAN-Tooling/manheim-jenkins-master) after bumping the ``jobsRepoRelease`` in [build/create_bootstrap_job.groovy](https://ghe.coxautoinc.com/MAN-Tooling/manheim-jenkins-master/blob/master/build/create_bootstrap_job.groovy#L8).

__See "Release Process" below for important information.__

## Groovy Script Debugging

Many of the Groovy methods used for ActiveChoice code generation include
``println`` debugging that will write to the Jenkins log (i.e. STDOUT, which is
exposed in Docker container logs).

To enable this: ``bundle exec rake spec:debugprint``

To disable this: ``bundle exec rake spec:noprint``

These commands will edit the source files.

## Testing

Bento-jobs has a variety of Rake-driven tests. Use ``bundle exec rake`` to view possible options.

__Note:__ The acceptance tests require some real credentials, including AWS credentials and a Vault token. See [spec/acceptance/jobs/README.md](spec/acceptance/jobs/README.md) for instructions.

* To run unit tests (Ruby/Rake stuff): ``bundle exec rake spec:unit``
* To run the (selenium/watir) acceptance tests: ``bundle exec rake spec:acceptance``
* To run __all__ tests: ``bundle exec rake spec:all``
* To run spec tests based on a specified file pattern: ``bundle exec rake spec:pattern[PATTERN]``

For in-depth information about the acceptance tests, see [spec/acceptance/jobs/README.md](spec/acceptance/jobs/README.md).

## Release Process

* __All Pull Requests__ must include an update to ``CHANGELOG.md``; see the comments at the top of the file for instructions.
* To get your PR merged to master _without_ triggering a new release, only modify ``CHANGELOG.md``
* __To trigger a new release__, increment the version in the ``version`` file according to the below instructions and ensure
  there is a ``CHANGELOG.md`` entry that matches the new version number and includes entries for all changes, as well as
  upgrade instructions for any actions users may need to take when upgrading. The release process is triggered by the
  ``version`` file containing a version number that doesn't match any of the tags on the git repo.

### Version Numbering

This project uses [Semantic Versioning](http://semver.org/). The short version of this is:

* Versions are in MAJOR.MINOR.PATCH format (i.e. ``x.y.z`` where each is an integer)
* Test (branch) builds are automatically suffixed by a dash followed by the git commit.
* Any changes that introduce breaking or non-backwards-compatible changes, which require
  or may require user action to upgrade to (including changes which may require corresponding
  changes to custom master images) must result in a __major version__ increment.
* Any changes that introduce backwards-compatible functionality or feature additions, which
  will not require any user changes at upgrade (including custom master images) should result
  in a __minor version__ increment.
* Any non-user-visible bug fixes that are completely backwards-compatible (including for
  custom master images) should result in a __patch version__ increment.

## Swarm Node Management Internals

The [jenkins-master container](https://ghe.coxautoinc.com/MAN-Tooling/manheim-jenkins-master) is started with ``/var/run/docker.sock`` bind mounted from the host, so that Jenkins can access the host's Docker daemon. In order to enable this, we explicitly create the ``docker`` group with GID 2001 on both the Bento host and in the manheim-jenkins-master image, and then add the ``jenkins`` user on both the host and the master to that group. The decision to handle the Docker swarm node management directly on the master was made for two reasons; first, this avoids the chicken-or-egg problem of needing to have an initial slave running to manage docker swarm slaves, and second, because most jobs are configured not to run on the master, teams' jobs shouldn't inadvertantly get access to the docker socket (they *can*, but it would almost certainly need to be intentional).

When a new Bento instances comes online, it has no swarm slaves. The slaves (currently only Docker implemented, but this could work for EC2 or anything else) are stored locally on disk in the master (``${JENKINS_HOME}/bento_swarm_nodes.json``) as a JSON hash of all slaves on a given Bento instance, including their name and all information required to summon them. When any of the slave management jobs are run on the Bento instance (``bento/bento_sync_swarm_nodes``, ``bento/bento_add_docker_swarm_node`` or ``bento/bento_remove_docker_swarm_node``) and the file doesn't exist, the jobs will use a default configuration containing one generic [generic-docker-swarm-node](https://ghe.coxautoinc.com/MAN-Containers/generic-docker-swarm-node).

All of these jobs are rake tasks which take their configuration via environment variables. The code behind the jobs is in ``lib/rake_helpers`` and should have near-100% rspec coverage. The jobs themselves (``config/bento_local_jobs.groovy``) are just thin wrappers around ``rake``. Note that the swarm-related jobs use environment variables which are set in Vagrant and the EC2 nodes at creation time to identify the Bento instance (``BENTO_TEAM_UID`` and ``BENTO_ENVIRONMENT``).

The ``add`` and ``remove`` jobs operate on the JSON hash on disk, adding and removing nodes to/from it. By default, on success, the jobs each trigger a run of ``bento/bento_sync_swarm_nodes`` to apply the changes (the current state stored on disk). If adding/removing multiple nodes, you can run the ``add`` and ``remove`` jobs with an option to not trigger sync afterwards.

The ``bento/bento_sync_swarm_nodes`` job retrieves the list of desired swarm nodes from the file on disk and queries the currently-running swarm nodes from Docker on the Bento host. It then pulls images and creates and starts containers for any swarm nodes that are in the list but not on the host, and then stops and removes any swarm node containers that are running on the host but not in the list. In order to make sure it does not affect containers not managed by the jobs (i.e. Datadog or the master itself), the jobs label containers with ``com.manheim.container.managed_by=bento-ruby`` and only operate on containers with that label.

The swarm state was originally stored in the re.consul.aws-dev.manheim.com Consul cluster. In version 1.1.0 when the work was done to decouple Bento from services in the manheim legacy AWS accounts, it was noted that swarm state (after being stored in Consul for over a year) was only ever referenced from these jobs on an individual Bento, and had never been utilized off the Bento instance itself. As such, the simplest method of storing swarm state was to store it in the JENKINS_HOME directory, where it's accessible by the master and will be backed up and restored along with the rest of the Bento backup.

## Builds of this Repository

### Jenkins Setup

Builds should be done via Jenkins. A ``Jenkinsfile`` is provided in this repository. To setup your Jenkins/Bento instance to use this, find an appropriate folder and [Define a new Pipeline from SCM](https://jenkins.io/doc/book/pipeline/getting-started/#defining-a-pipeline-in-scm):

* ``New Item`` -> ``Multibranch Pipeline``
  * ``Build Configuration`` -> ``by Jenkinsfile``
  * ``Branch Sources`` -> ``Add Source`` -> ``GitHub``
    * fill in the GitHub repository information
    * be sure to select credentials with __write__ access
    * Click the "Advanced" button under the Repository dropdown
    * Ensure that __only__ the following "^Build" options are checked:
      * Build origin branches
      * Build origin PRs (merged with base branch)
      * Build fork PRs (merged with base branch)

*(Note: jantman spent 8+ hours trying to automate this via the Jenkinsfile, but that doesn't appear to currently be possible as of 2017-05-23)*

### GitHub Setup

1. Add a webhook on the repo to ``$JENKINS_URL/github-webhook/`` for Push, Pull Request and Repository events.
2. Open a PR; this can be a throw-away PR that you close after, but you just need something to trigger the Jenkins PR build.
3. On the Repository Settings -> Branches page, under "Protected Branches", choose "master" from the dropdown. On the next page, check off:
   * Protect this Branches
   * Require pull request reviews before merging
     * Include administrators
   * Require status checks to pass before merging
     * Include administrators
   * Require branches to be up to date before merging
   * Check off the "continuous-integration/jenkins/pr-merge" status check
