# Percy::Capybara [<i class="fa fa-github" aria-hidden="true"></i>](https://github.com/percy/percy-capybara)

[![](https://travis-ci.org/percy/percy-capybara.svg?branch=master)](https://travis-ci.org/percy/percy-capybara)
[![](https://badge.fury.io/rb/percy-capybara.svg)](https://rubygems.org/gems/percy-capybara)

Percy::Capybara is a library for integrating Percy into your existing [Capybara](https://github.com/jnicklas/capybara) feature tests in any Ruby web framework—including Rails, Sinatra, etc.

If you've written feature tests (or "UI / acceptance / browser tests"), you know how hard it can be to get them right and to get your app in the correct UI state. Percy::Capybara lets you take all the time you've spent building your feature tests and expand them with screenshots and visual testing at every step of the way, to truly see what the browser sees.

The examples below assume you are using RSpec, but they could be easily adapted for other testing frameworks.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'percy-capybara'
```

And then execute:

```bash
$ bundle install
```

Or install it yourself with `gem install percy-capybara`.

## Setup

Test suites need to call `Percy::Capybara.initialize_build` before running and
`Percy::Capybara.finalize_build` when finished (even if your tests fail).

<div class="Alert Alert--warning">
  <strong>NOTE:</strong> If your builds get stuck <i>"Receiving..."</i>, check this configuration
  first.
</div>

### RSpec

Add these setup lines to `spec_helper.rb`:

```ruby
RSpec.configure do |config|
  # ...

  config.before(:suite) { Percy::Capybara.initialize_build }
  config.after(:suite) { Percy::Capybara.finalize_build }
end
```

Note that RSpec runs before hooks in order, so these lines must come after any Percy authentication
setup done in private repositories in `before(:suite)` hooks.

### Other test frameworks (MiniTest, etc.)

For example, with MiniTest you can add this to your `test_helper.rb`:

```ruby
Percy::Capybara.initialize_build MiniTest.after_run { Percy::Capybara.finalize_build }
```

### Non-Rails frameworks (Sinatra, etc.)

Without Rails autoloading, you will need to manually `require 'percy/capybara'` when using Percy.

## Usage

Now the fun part!

You can integrate with Percy by adding one line to your existing feature specs:

```ruby
describe 'a feature', type: :feature, js: true do
  it 'shows the dropdown menu when clicked' do
    visit '/'
    first('.dropdown-toggle').click
    expect(page).to have_selector('#main-dropdown', visible: true)

    Percy::Capybara.snapshot(page, name: 'homepage with dropdown')
  end
end
```

The `name: 'homepage with dropdown'` argument is not required, but it is helpful to identify the page by more than just its URL. If you are snapshotting a page multiple times with the same URL, `name` must be set. See _Identifying snapshots_ below.

Done! A new Percy build will be created the next time you run your tests in a supported CI service, or it can be triggered locally.

### Local dev environments

By default, Percy is disabled in local dev environments to avoid teams accidentally overriding each others' changes while developing feature specs. However, you may want to enable Percy locally while getting set up for the first time. Or, you may want to see visual diffs as you iterate on a design but before pushing the final changes for review.

You can enable Percy for your local environment by setting the `PERCY_ENABLE` environment variable,
and  temporarily setting the `PERCY_TOKEN` environment variable locally:

```bash
$ export PERCY_ENABLE=1  # Required only in local dev environments.
$ export PERCY_TOKEN=1234abcd1234abcd
$ bundle exec rspec
```

Or in one line:

```bash
$ PERCY_ENABLE=1 PERCY_TOKEN=1234abcd1234abcd bundle exec rspec
```

Careful though—if you run this in your local `master` branch, Percy cannot tell the difference between your local environment and your CI environment, so this will set the repo's `master` state in Percy. You can avoid this by simply checking out a different branch, or setting the `PERCY_BRANCH` environment variable.


### Responsive visual diffs

(WRITEME)

### Identifying snapshots

Percy needs to be able to uniquely identify the same snapshot across builds to provide visual diffs.

To accomplish this, `Percy::Capybara` uses the current page URL as the name of the snapshot. We assume that your app has been built with stateful navigation and that the URL fully identifies the page state.

However, there are many cases where this is not enough—for example, populating test data in the page, or performing a UI interaction that doesn't change the URL (like clicking the dropdown in the example above).

To manually identify a snapshot, you can provide the `name` parameter:

```ruby
Percy::Capybara.snapshot(page, name: 'homepage (with dropdown clicked)')
```

The `name` param can be any string that makes sense to you to identify the page state, it should just be unique and remain the same across builds.

It is **required** if you are snapshotting a page multiple times with the same URL.

## How it works

The actual page rendering and diffing, which often can be very slow and computationally expensive, does _not_ happen in your tests. Instead, the `snapshot` method grabs a the current DOM structure, CSS, and the page's assets and uploads them to Percy.

Percy then handles all the complexities of rendering the page in a modern browser (Firefox 38 ESR), taking a screenshot, generating a pixel-by-pixel visual diff compared to the last successful master build, setting the status of the GitHub Pull Request, etc. This is performed in our custom, parallelized rendering infrastructure—built specifically to support processing and storing hundreds or thousands of snapshots at the same time, as fast as your team and app size may need.

This asset-centric architecture keeps the impact on your tests minimal—in most cases, the overhead for each snapshot in your tests is only a few milliseconds. Assets are also fingerprinted when uploaded, so they are only uploaded once.

## Troubleshooting

### WebMock/VCR users

If you use [VCR](https://github.com/vcr/vcr) to mock and record HTTP interactions, you need to allow connections to the Percy API:

```ruby
VCR.configure do |config|
  config.ignore_hosts 'percy.io'
end
```

If you use [WebMock](https://github.com/bblimke/webmock) to stub out HTTP connections, you need to allow connections to the Percy API:

```ruby
WebMock.disable_net_connect!(allow: 'percy.io')
```

### Turn off debug assets

After upgrading to Sprockets 3, you may notice broken CSS in Percy builds. You likely have this option set in `test.rb`:

```ruby
config.assets.debug = true
```

This must be set to false in your `test.rb` file:

```ruby
config.assets.debug = false
```

There is no compelling reason to have debug assets permanently enabled in tests—debug assets disables concatination of asset files and will negatively affect your test performance and consistency. You must turn off debug assets in tests for Percy to work correctly.

## Other resources

*   [Percy::Capybara Reference](http://www.rubydoc.info/gems/percy-capybara/Percy/Capybara) on RubyDoc

## Changelog

*   [percy-capybara releases on GitHub](https://github.com/percy/percy-capybara/releases)

## Contributing

1.  Fork it ([https://github.com/percy/percy-capybara/fork](https://github.com/percy/percy-capybara/fork))
2.  Create your feature branch (`git checkout -b my-new-feature`)
3.  Commit your changes (`git commit -am 'Add some feature'`)
4.  Push to the branch (`git push origin my-new-feature`)
5.  Create a new Pull Request