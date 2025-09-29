# Installing Foreman from Source on Ubuntu

This guide describes how to run the Foreman installer directly from this
repository on a fresh Ubuntu 20.04 or 22.04 server. The steps mirror what the
packaged installer does, but give you full control over the source tree.

## 1. Prepare the operating system

1. Update the package index and install the base toolchain and Ruby headers.

   ```bash
   sudo apt-get update
   sudo apt-get install -y build-essential git ruby ruby-dev zlib1g-dev libsqlite3-dev
   ```

2. Install Bundler so Ruby gems can be installed in the project directory.

   ```bash
   sudo gem install bundler --no-document
   ```

3. Enable the official Foreman APT repositories so the installer can download
   Foreman, Foreman Proxy, and any plugin packages. Replace `jammy` with
   `focal` if you are using Ubuntu 20.04.

   ```bash
   wget https://apt.theforeman.org/foreman.asc
   sudo mv foreman.asc /usr/share/keyrings/foreman-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/foreman-archive-keyring.gpg] http://deb.theforeman.org/ jammy stable" | \
     sudo tee /etc/apt/sources.list.d/foreman.list
   echo "deb [signed-by=/usr/share/keyrings/foreman-archive-keyring.gpg] http://deb.theforeman.org/ plugins stable" | \
     sudo tee /etc/apt/sources.list.d/foreman-plugins.list
   sudo apt-get update
   ```

## 2. Clone the installer and install Ruby dependencies

```bash
cd /opt
sudo git clone https://github.com/theforeman/foreman-installer.git
sudo chown -R "$USER" foreman-installer
cd foreman-installer
bundle config set --local path 'vendor/bundle'
bundle install
```

The `bundle install` step pulls in Kafo, Puppet, and the other Ruby gems that
power the installer.

## 3. Download the Puppet modules required by the installer

The installer relies on a large collection of Puppet modules listed in the
project `Puppetfile`. Use the bundled Rake task to download them into the local
build directory:

```bash
bundle exec rake modules
```

When this finishes you should see the modules under `_build/modules`.

## 4. Review or customize installer answers

Each installer scenario (Foreman, Foreman Proxy Content, Katello) ships with an
answer file in `config/*.yaml`. For a simple all-in-one Foreman install, copy the
provided defaults into `/etc/foreman-installer/scenarios.d/` and adjust any
settings you need (for example to point to an external database or set a custom
administrator password):

```bash
sudo install -d /etc/foreman-installer/scenarios.d
sudo cp config/foreman-answers.yaml /etc/foreman-installer/scenarios.d/
```

The files are standard YAML. Set a parameter to `true` to enable the matching
component, or override individual settings under the relevant key. You can leave
the defaults in place and rely on the interactive installer in the next step if
you prefer.

## 5. Run the installer

Execute the installer under Bundler so it uses the gems you just installed:

```bash
sudo bundle exec bin/foreman-installer --scenario foreman
```

The installer will configure Foreman, Apache, PostgreSQL, and Puppet services.
If you want to walk through the options interactively, add `-i`:

```bash
sudo bundle exec bin/foreman-installer --scenario foreman -i
```

When the command completes successfully you can log in at
`https://<your-server.example.com>/` using the credentials shown at the end of
the run.

## 6. Keeping the installation up to date

* To apply updates to the Puppet modules, re-run `bundle exec rake modules`.
* To update Ruby gems, pull the latest changes and run `bundle install`.
* Re-run the installer whenever you modify the answer file so configuration
  drifts are corrected.

## Troubleshooting tips

* The installer logs are written to `/var/log/foreman-installer/`. Check these if
  Puppet runs fail.
* Puppet module downloads require outbound HTTPS access to the Forge.
* If Bundler installs gems into the system Ruby by mistake, ensure you ran the
  commands in the cloned repository and that the `bundle config set --local path`.
