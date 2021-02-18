+++
title="Adding Bill-of-Materials"
weight=409
+++

<!-- test:suite=create-buildpack;weight=9 -->

One of the benefits of buildpacks is they can also populate the app image with metadata from the build process, allowing you to audit the app image for information like:

* The process types that are available and the commands associated with them
* The run-image the app image was based on
* The buildpacks were used to create the app image
* And more...!

You can find all of this information using `pack` via its `inspect-image` command.


<!-- test:exec -->
```bash
pack inspect-image test-ruby-app
```
You should see the following:

<!-- test:assert=contains -->
```text
Run Images:
  cnbs/sample-stack-run:bionic

Buildpacks:
  ID                                  VERSION
  com.examples.buildpacks.ruby        0.0.1

Processes:
  TYPE                 SHELL        COMMAND                        ARGS
  web (default)        bash         bundle exec ruby app.rb        
  worker               bash         bundle exec ruby worker.rb
```

Apart from the above standard metadata, buildpacks can also populate information about the dependencies they have provided in form of a `Bill-of-Materials`. Let's see how we can use this to populate information about the version of `ruby` that was installed in the output app image.

To add the `ruby` version to the output of `pack inspect-image`, we will have to provide a `Bill-of-Materials` (`BOM`) containing this information. You'll need to update the `launch.toml` created at the end of your `build` script:

```bash
# ...

# Append a Bill-of-Materials containing metadata about the provided ruby version
cat >> "$layersdir/launch.toml" <<EOL
[[bom]]
name = "ruby"
[bom.metadata]
version = "$ruby_version"
EOL
```

Your `ruby-buildpack/bin/build` script should look like the following:

<!-- test:file=ruby-buildpack/bin/build -->
```bash
#!/usr/bin/env bash
set -eo pipefail

echo "---> Ruby Buildpack"

# 1. GET ARGS
layersdir=$1
plan=$3

# 2. DOWNLOAD RUBY
rubylayer="$layersdir"/ruby
mkdir -p "$rubylayer"
ruby_version=$(cat "$plan" | yj -t | jq -r '.entries[] | select(.name == "ruby") | .metadata.version')
echo "---> Downloading and extracting Ruby $ruby_version"
ruby_url=https://s3-external-1.amazonaws.com/heroku-buildpack-ruby/heroku-18/ruby-$ruby_version.tgz
wget -q -O - "$ruby_url" | tar -xzf - -C "$rubylayer"

# 3. MAKE RUBY AVAILABLE DURING LAUNCH
echo -e 'launch = true' > "$rubylayer.toml"

# 4. MAKE RUBY AVAILABLE TO THIS SCRIPT
export PATH="$rubylayer"/bin:$PATH
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH:+${LD_LIBRARY_PATH}:}"$rubylayer/lib"

# 5. INSTALL BUNDLER
echo "---> Installing bundler"
gem install bundler --no-ri --no-rdoc

# 6. INSTALL GEMS
# Compares previous Gemfile.lock checksum to the current Gemfile.lock
bundlerlayer="$layersdir/bundler"
local_bundler_checksum=$((sha256sum Gemfile.lock >/dev/null 2>&1 || echo 'DOES_NOT_EXIST') | cut -d ' ' -f 1)
remote_bundler_checksum=$(cat "$bundlerlayer.toml" | yj -t | jq -r .metadata.checksum 2>/dev/null || echo 'DOES_NOT_EXIST')

if [[ -f Gemfile.lock && $local_bundler_checksum == $remote_bundler_checksum ]] ; then
    # Determine if no gem dependencies have changed, so it can reuse existing gems without running bundle install
    echo "---> Reusing gems"
    bundle config --local path "$bundlerlayer" >/dev/null
    bundle config --local bin "$bundlerlayer/bin" >/dev/null
else
    # Determine if there has been a gem dependency change and install new gems to the bundler layer; re-using existing and un-changed gems
    echo "---> Installing gems"
    mkdir -p "$bundlerlayer"
    cat > "$bundlerlayer.toml" <<EOL
cache = true
launch = true

[metadata]
checksum = "$local_bundler_checksum"
EOL
    bundle config set --local path "$bundlerlayer" && bundle install && bundle binstubs --all --path "$bundlerlayer/bin"

fi

# 7. SET DEFAULT START COMMAND
cat > "$layersdir/launch.toml" <<EOL
# our web process
[[processes]]
type = "web"
command = "bundle exec ruby app.rb"

# our worker process
[[processes]]
type = "worker"
command = "bundle exec ruby worker.rb"
EOL

# 8. ADD A BOM
cat >> "$layersdir/launch.toml" <<EOL
[[bom]]
name = "ruby"
[bom.metadata]
version = "$ruby_version"
EOL

```

Then rebuild your app using the updated buildpack:

<!-- test:exec -->
```bash
pack build test-ruby-app --path ./ruby-sample-app --buildpack ./ruby-buildpack
```

You should then be able to inspect your Ruby app for its Bill-of-Materials via:

<!-- test:exec -->
```bash
pack inspect-image test-ruby-app --bom
```

You should find that the included `ruby` version is `2.5.0` as expected.

<!-- test:assert=contains -->
```text
    {
      "name": "ruby",
      "metadata": {
        "version": "2.5.0"
      },
      "buildpacks": {
        "id": "com.examples.buildpacks.ruby",
        "version": "0.0.1"
      }
    }
```

Congratulations! You’ve created your first configurable Cloud Native Buildpack that uses detection, image layers, and caching to create an introspectable and runnable OCI image.

## Going further

Now that you've finished your buildpack, how about extending it? Try:

- Caching the downloaded Ruby version
- Updating the BOM with all the gems provided by bundler
- [Packaging your buildpack for distribution][package-a-buildpack]

[package-a-buildpack]: /docs/buildpack-author-guide/package-a-buildpack/