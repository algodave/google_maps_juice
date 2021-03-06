# GoogleMapsJuice

**Drink Google Maps Services with ease!** :tropical_drink: :earth_americas:

[![Build Status](https://travis-ci.org/algonauti/google_maps_juice.svg?branch=master)](https://travis-ci.org/algonauti/google_maps_juice)
[![Coverage Status](https://coveralls.io/repos/github/algonauti/google_maps_juice/badge.svg?branch=master)](https://coveralls.io/github/algonauti/google_maps_juice?branch=master)

This gem aims at progressively covering a fair amount of those widely-used services that are part of the Google Maps Platform, such as: [Geocoding](https://developers.google.com/maps/documentation/geocoding/intro), [Time Zone](https://developers.google.com/maps/documentation/timezone/intro), [Directions](https://developers.google.com/maps/documentation/directions/intro), etc. with some key ideas:

1. Allowing "standard" requests, meaning: sending the same params documented by Google.
2. Allowing "smart" requests, meaning: with more "developer-friendly" params, and/or improved error handling.
3. Return full Google responses, but also provide methods to easily inspect the most relevant info.
4. Provide error handling.


`GoogleMapsJuice` currently covers:

* Geocoding
* Time Zone
* Directions

Contributors are welcome!


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'google_maps_juice'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install google_maps_juice


## Google API Key

You can set your Google API key with the following one-liner:

```ruby
GoogleMapsJuice.configure { |c| c.api_key = 'my-api-key' }`
```

In a Rails application, that would typically go in an initializer.


### Multiple API keys

If you need to use multiple API keys, you have two options:

* a) pass an `api_key` named param to the endpoint class method, e.g.

```ruby
GoogleMapsJuice::Geocoding.geocode(params, api_key: 'my-api-key')
```

* b) create your own `GoogleMapsJuice::Client` instance(s) and use it to create your endpoint object(s), e.g.

```ruby
client = GoogleMapsJuice::Client.new(api_key: 'my-api-key')
geocoding = GoogleMapsJuice::Geocoding.new(client)
response = geocoding.geocode(params)
```

This is especially useful in some "hybrid" scenario, where an API key is shared by a group of requests, but another group uses a different key: a `client` object would then be instantiated and reused for each group.


## Error Handling

If Google servers respond with a non-successful HTTP status code, i.e. `4xx` or `5xx`, a `GoogleMapsJuice::ResponseError` is raised with a message of the form `'HTTP 503 - Error details as returned by the server'`.

API errors are also handled, based on the `status` attribute of Google's JSON response, and the optional `error_message` attribute.

* `GoogleMapsJuice::ZeroResults` is raised when `status` is `'ZERO_RESULTS'`
* `GoogleMapsJuice::ApiLimitError` is raised when `status` is `'OVER_DAILY_LIMIT'` or `'OVER_QUERY_LIMIT'`
* `GoogleMapsJuice::ResponseError` is raised when `status` is not `OK` with a message of the form `API <status> - <error_message>`


## Geocoding

### Standard Geocoding

The simplest geocoding requests accept an address:

```ruby
response = GoogleMapsJuice::Geocoding.geocode(address: '8955 Lantana Rd, Lake Worth, FL 33467, USA')
```

Supported params are the ones accepted by Google's endpoint: `address`, `components`, `bounds`, `language`, `region`; at least one between `address` and `components` is required. Learn more [here](https://developers.google.com/maps/documentation/geocoding/intro#geocoding). `GoogleMapsJuice` will raise an `ArgumentError` if some unsupported param is passed, or when none of the required params are passed.


### Smart Geocoding

**Motivation**

For best geocoding results, the `address` param should be formatted according to the local language. This is often a hard task for an application that needs to geocode addresses stored as separate fields. Luckily, Google offers the `components` param which accepts individual address fields; however, it's annoying to build it and it's not fault tolerant. For example, an error on `postal_code` makes geocoding of a whole address fail.

Purpose of `i_geocode` method is twofold:

1. Providing a simpler method interface for leveraging Google's `components` param
2. Providing an approximate geocoding result when some address component is wrong

Here an example call with all supported params:

```ruby
response = GoogleMapsJuice::Geocoding.i_geocode(
  {
    address: '8955 Lantana Rd',
    locality: 'Lake Worth',
    administrative_area: 'FL',
    postal_code: '33467',
    country: 'US'
  }, sleep_before_retry: 0.15
)
```

**Accepted params:**

* At least one between `address` and `country` is required
* `locality`, `administrative_area`, `postal_code` and `country` expect the same content as described in [Component Filtering](https://developers.google.com/maps/documentation/geocoding/intro#ComponentFiltering)
* `address` can also include more info than street number and name, as long as they do not contrast with other params passed
* An optional `sleep_before_retry` param sets seconds between geocoding attempts (see below); defaults to zero.
* `GoogleMapsJuice` will raise an `ArgumentError` if some unsupported param is passed, or when none of the required params are passed.

**How it works**

On its 1st attempt, `i_geocode` sends all received params to Google's endpoint, properly formatted. If a `GoogleMapsJuice::ZeroResults` is raised, it removes a param and retries until no error is raised. Params are removed in the following order:

* `postal_code`
* `address`
* `locality`
* `administrative_area`

As a consequence:

* In the best case, `i_geocode` will send 1 request to Google API
* In the worst case, `i_geocode` will send 4 requests to Google API


### Geocoding Response

Both `geocode` and `i_geocode` methods return a `GoogleMapsJuice::Geocoding::Response`. It's a `Hash` representation of Google's JSON response. However, it also provides many useful methods:

* `latitude`, `longitude`: geographic coordinates as `float` numbers

* `street_number`, `route`, `locality`, `postal_code`, `administrative_area_level_1`, `country`: all of these methods return a `Hash` with 2 keys: `'short_name'` and `'long_name'`

* `partial_match?`: boolean, `true` if some param (of the last geocoding attempt) partially matched

* `precision`: can be one of: `'street_number'`, `'route'`, `'locality'`, `'postal_code'`, `'administrative_area_level_1'`, `'country'` and represents the most-specific matching component


## Time Zone

[Google's Time Zone API](https://developers.google.com/maps/documentation/timezone/intro#Requests) returns the time zone of a given geographic location; it also accepts a timestamp, in order to determine whether DST should be applied or not.

GoogleMapsJuice provides the `GoogleMapsJuice::Timezone.by_location` method. Compared to Google's raw API request, it provides simpler params and some validations, in order to avoid sending requests when they would fail for sure (and then save money!) - to learn more see `spec/unit/timezone_spec.rb`.

**Accepted params:**

* Both `latitude` and `longitude` are mandatory
* `timestamp` is optional and defaults to `Time.now`
* `language` is optional


### Time Zone Response

The `by_location` method returns a `GoogleMapsJuice::Timezone::Response`. It's a `Hash` representation of Google's JSON response. However, it also provides a few useful methods:

* `timezone_id`: unique name as defined in [IANA Time Zone Database](https://www.iana.org/time-zones)

* `timezone_name`: the long form name of the time zone

* `raw_offset`: the offset from UTC in seconds

* `dst_offset`: the offset for daylight-savings time in seconds


## Directions

[Google's Directions API](https://developers.google.com/maps/documentation/directions/intro#DirectionsRequests) returns the possible routes of given origin and destination geographic locations; Google's API accepts address, textual latitude/longitude value, or place ID of which you wish to calculate directions. Currently this gem implements only latitude/longitude mode.
`GoogleMapsJuice` will raise an `ArgumentError` if some unsupported param is passed, or when none of the required params are passed.

```ruby
response = GoogleMapsJuice::Directions.find(origin: '41.8892732,12.4921921', destination: '41.9016488,12.4583003')
```

Compared to Google's raw API request, it provides validation of both origin and destination, in order to avoid sending requests when they would fail for sure - to learn more see `spec/unit/directions_spec.rb`.

**Accepted params:**

* Both `origin` and `destination` are mandatory
* `origin` is composed by `latitude` and `longitude`, comma separated float values
* `destination` same format as `origin`


### Directions Response

The `find` method returns a `GoogleMapsJuice::Directions::Response`. It's a `Hash` representation of Google's JSON response. However, it also provides a few useful methods:

* `results`: the `Hash` raw result

* `routes`: an `Array` of `GoogleMapsJuice::Directions::Response::Route` objects

* `first`: the first `Route` of the `routes` `List`

As described in [Google's Directions API](https://developers.google.com/maps/documentation/directions/intro#Routes), the response contains all possible routes. Each route has some attributes and one or more *legs*, wich in turn have one or more *steps*.
If no waypoints are passed, the route response will contain a single leg. Since `GoogleMapsJuice::Directions` doesn't handles waypoints yet, only the first leg is considered for each route.
The `GoogleMapsJuice::Directions::Response::Route` is a representation of a response route and provides methods to access all route's attributes:

* `summary`: a brief description of the route

* `legs`: all legs of the route, generally a single one

* `steps`: all steps of the first route's leg

* `duration`: time duration of the first route's leg

* `distance`: distance between origin and destination of first route's leg

* `start_location`: `latitude`/`longitude` of the origin first route's leg

* `end_location`: `latitude`/`longitude` of the destination first route's leg

* `start_address`: address of the origin first route's leg

* `end_address`: address of the destination first route's leg


## Development

After checking out the repo, run `bin/setup` to install dependencies. Create a `.env` file and save your Google API key there; if you want to use a different key for testing, put it in `.env.test` and it will override the one in `.env`.

Run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags and push the `.gem` file to [rubygems.org](https://rubygems.org).


### Implementing a new endpoint

All endpoints must be subclasses of `GoogleMapsJuice::Endpoint`; methods that implement "standard" Google API calls have a common structure, described by the `invoke` method in the `SomeEndpoint` test class in `spec/unit/endpoint_spec.rb`.

All new endpoints' methods must return subclasses of `GoogleMapsJuice::Endpoint::Response` as their response objects, since it contains methods needed for error handling.


## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/algonauti/google_maps_juice.


## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
