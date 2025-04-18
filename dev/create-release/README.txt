Entrance script is _do-release-docker.sh_. Requires a local docker;
for example, on macOS, Docker for Desktop installed and running.

For usage, pass '-h':

 $ ./do-release-docker.sh -h

./do-release-docker.sh accepts the following options

  -d [path]    required. working directory. output will be written to "output" in here.
  -f           "force" -- actually publish this release. Unless you specify '-f', it will
               default to dry run mode, which checks and does local builds, but does not upload anything.
  -t [tag]     tag for the phoenix-rm docker image to use for building (default: "latest").
  -j [path]    path to local JDK installation to use building. By default the script will
               use openjdk8 installed in the docker image.
  -p [project] project to build, such as 'phoenix' or 'phoenix-connectors'; defaults to phoenix env var
  -r [repo]    git repo to use for remote git operations. defaults to ASF gitbox for project.
  -s [step]    runs a single step of the process; valid steps are: tag|publish-dist|publish-release.
               If none specified, runs tag, then publish-dist, and then publish-release.
               'publish-snapshot' is also an allowed, less used, option.
  -h           display usage information
  -x           debug. do less clean up. (env file, gpg forwarding on mac)

For example, use the following command do a full dry run build of phoenix-thirdparty:

./do-release-docker.sh -d /tmp/thirdparty-build -p phoenix-thirdparty

To run a build w/o invoking docker (not recommended!), use _do_release.sh_.

Both scripts will query interactively for needed parameters and passphrases.
For explanation of the parameters, execute:
 $ release-build.sh --help

Before starting the RC build, run a reconciliation of what is in
JIRA with what is in the commit log. Make sure they align and that
anomalies are explained up in JIRA.

See http://hbase.apache.org/book.html#maven.release
(Even though the above documentation is for HBase, we use the same process for Phoenix.)

Regardless of where your release build will run (locally, locally in docker, on a remote machine,
etc) you will need a local gpg-agent with access to your secret keys. A quick way to tell gpg
to clear out state and start a gpg-agent is via the following command phrase:

$ gpgconf --kill all && gpg-connect-agent /bye

Before starting an RC build, make sure your local gpg-agent has configs
to properly handle your credentials, especially if you want to avoid
typing the passphrase to your secret key.

e.g. if you are going to run and step away, best to increase the TTL
on caching the unlocked secret via ~/.gnupg/gpg-agent.conf
  # in seconds, e.g. a day
  default-cache-ttl 86400
  max-cache-ttl 86400

In the current version, passphrase entry doesn't work at all, at least for Linux Docker builds.
Increasing the TTL only works if you unlock the key before starting the release script by running
gpg separately before the script.
A better way to handle passphrases without changing the TTLs is to preset the passphrase,
which avoids using pinentry mechanism completely, and will be reset on logout.

# Find the "gpg-preset-passphrase" program. It is not on the PATH by default.
$ find / -name gpg-preset-passphrase
# Make sure you have the "allow-preset-passphrase" line  in your $HOME/.gnupg/gpg-agent.conf
# Restart gpg
$ gpgconf --kill all && gpg-connect-agent /bye
# List your keys with key grip
$ gpg --with-keygrip --list-secret-keys
# Preset the passphrase for your signing key
# </full/path/to/>/gpg-preset-passphrase -P <the passphrase> -c <the keygrip>
# Check that the passphrase is successfully preset. There should be a '1' at the fourth position
# after the keygrip for your key in the output for the signing key
$ gpg-connect-agent 'keyinfo --list' /bye
# Run the release script (see above)
# Restart the gpg agent again to make sure it forgets the preset passphrase
$ gpgconf --kill all && gpg-connect-agent /bye

Running a build on GCE is easy enough. Here are some notes if of use.
Create an instance. 4CPU/15G/10G disk seems to work well enough.
Once up, run the below to make your machine fit for RC building:

Note that according to https://www.apache.org/legal/release-policy.html#owned-controlled-hardware
Building an Apache release must be done on hardware owned and controlled by the committer.

# Presuming debian-compatible OS, do these steps on the VM
# your VM username should be your ASF id, because it will show up in build artifacts.
# Follow the docker install guide: https://docs.docker.com/engine/install/debian/
$ sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
$ sudo add-apt-repository -y \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
$ sudo apt-get update
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io
# Follow the post installation steps: https://docs.docker.com/engine/install/linux-postinstall/
$ sudo usermod -aG docker $USER
# LOGOUT and then LOGIN again so $USERID shows as part of docker group
# Test here by running docker's hello world as your build user
$ docker run hello-world

# Follow the GPG guide for forwarding your gpg-agent from your local machine to the VM
#   https://wiki.gnupg.org/AgentForwarding
# On the VM find out the location of the gpg agent socket and extra socket
$ gpgconf --list-dir agent-socket
/run/user/1000/gnupg/S.gpg-agent
$ gpgconf --list-dir agent-extra-socket
/run/user/1000/gnupg/S.gpg-agent.extra
# On the VM configure sshd to remove stale sockets
$ sudo bash -c 'echo "StreamLocalBindUnlink yes" >> /etc/ssh/sshd_config'
$ sudo systemctl restart ssh
# logout of the VM

# Do these steps on your local machine.
# make sure gpg-agent is running
$ gpg-connect-agent /bye
# Export your public key and copy it to the VM.
# Assuming 'example.gce.host' maps to your VM's external IP (or use the IP)
$ gpg --export example@apache.org > ~/gpg.example.apache.pub
$ scp ~/gpg.example.apache.pub example.gce.host:
# ssh into the VM while forwarding the remote gpg socket locations found above to your local
#   gpg-agent's extra socket (this will restrict what commands the remote node is allowed to have
#   your agent handle. Note that the gpg guide above can help you set this up in your ssh config
#   rather than typing it in ssh like this every time.
# Note that as of maven-gpg-plugin 3.0.1, with gnupg >= 2.1, the plugin uses
#   `--pinentry-mode error`, which is apparently not supported over the `extra` socket. These
#   instructions may require tweaking.
$ ssh -i ~/.ssh/my_id \
    -R "/run/user/1000/gnupg/S.gpg-agent:$(gpgconf --list-dir agent-extra-socket)" \
    -R "/run/user/1000/gnupg/S.gpg-agent.extra:$(gpgconf --list-dir agent-extra-socket)" \
    example.gce.host

# now in an SSH session on the VM with the socket forwarding
# import your public key and test signing with the forwarding to your local agent.
$ gpg --no-autostart --import gpg.example.apache.pub
$ echo "foo" > foo.txt
$ gpg --no-autostart --detach --armor --sign foo.txt
$ gpg --no-autostart --verify foo.txt.asc

# install git and clone the main project on the build machine
$ sudo apt-get install -y git
$ git clone https://github.com/apache/phoenix.git
# finally set up an output folder and launch a dry run.
$ mkdir ~/build
$ cd phoenix
$ ./dev/create-release/do-release-docker.sh -d ~/build

# for building the main repo specifically you can save an extra download by pointing the build
# to the local clone you just made
$ ./dev/create-release/do-release-docker.sh -d ~/build -r .git
