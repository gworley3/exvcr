# ExVCR [![Build Status](https://secure.travis-ci.org/parroty/exvcr.png?branch=master "Build Status")](https://travis-ci.org/parroty/exvcr) [![Coverage Status](https://coveralls.io/repos/parroty/exvcr/badge.png?branch=master)](https://coveralls.io/r/parroty/exvcr?branch=master) [![hex.pm version](https://img.shields.io/hexpm/v/exvcr.svg)](https://hex.pm/packages/exvcr) [![hex.pm downloads](https://img.shields.io/hexpm/dt/exvcr.svg)](https://hex.pm/packages/exvcr) [![License](https://img.shields.io/badge/license-MIT-brightgreen.svg)](http://opensource.org/licenses/MIT)


Record and replay HTTP interactions library for elixir.
It's inspired by Ruby's VCR (https://github.com/vcr/vcr), and trying to provide similar functionalities.

### Basics

- The following HTTP libraries can be applied.
    - <a href="https://github.com/cmullaparthi/ibrowse" target="_blank">ibrowse</a>-based libraries.
        - <a href="https://github.com/myfreeweb/httpotion" target="_blank">HTTPotion</a>
    - <a href="https://github.com/benoitc/hackney" target="_blank">hackney</a>-based libraries.
        - <a href="https://github.com/edgurgel/httpoison" target="_blank">HTTPoison</a>
        - support is very limited, and tested only with sync request of HTTPoison yet.
    - <a href="http://erlang.org/doc/man/httpc.html" target="_blank">httpc</a>-based libraries.
        - <a href="https://github.com/tim/erlang-oauth/" target="_blank">erlang-oauth</a>
        - <a href="https://github.com/Zatvobor/tirexs" target="_blank">tirexs</a>
        - support is very limited, and tested only with :httpc.request/1 and :httpc.request/4

- HTTP interactions are recorded as JSON file.
    - The JSON file can be recorded automatically (vcr_cassettes) or manually updated (custom_cassettes)

### Notes
- In case test behaves unstable, please try to specify `use ExUnit.Case, async: false`.

### Install
Add `:exvcr` to `deps` section of `mix.exs`.

```elixir
  def deps do
    [ {:exvcr, "~> 0.8", only: :test} ]
  end
```

Optionally, `preferred_cli_env: [vcr: :test]` can be specified for running `mix vcr` in `:test` env by default.

```elixir
  def project do
    [ ...
      preferred_cli_env: [
        vcr: :test, "vcr.delete": :test, "vcr.check": :test, "vcr.show": :test
      ],
      ...
  end
```

### Usage
- Add `use ExVCR.Mock` to the test module. This mocks ibrowse by default. For using hackney, specify `adapter: ExVCR.Adapter.Hackney` options as follows.

##### Example with ibrowse
```Elixir
defmodule ExVCR.Adapter.IBrowseTest do
  use ExUnit.Case, async: false
  use ExVCR.Mock

  setup_all do
    ExVCR.Config.cassette_library_dir("fixture/vcr_cassettes")
    :ok
  end

  test "example single request" do
    use_cassette "example_ibrowse" do
      :ibrowse.start
      {:ok, status_code, _headers, body} = :ibrowse.send_req('http://example.com', [], :get)
      assert status_code == '200'
      assert to_string(body) =~ ~r/Example Domain/
    end
  end

  test "httpotion" do
    use_cassette "example_httpotion" do
      HTTPotion.start
      assert HTTPotion.get("http://example.com", []).body =~ ~r/Example Domain/
    end
  end
end
```

##### Example with hackney
```Elixir
defmodule ExVCR.Adapter.HackneyTest do
  use ExUnit.Case, async: false
  use ExVCR.Mock, adapter: ExVCR.Adapter.Hackney

  setup_all do
    HTTPoison.start
  end

  test "get request" do
    use_cassette "httpoison_get" do
      assert HTTPoison.get!("http://example.com").body =~ ~r/Example Domain/
    end
  end
end
```

##### Example with httpc
```Elixir
defmodule ExVCR.Adapter.HttpcTest do
  use ExUnit.Case, async: false
  use ExVCR.Mock, adapter: ExVCR.Adapter.Httpc

  setup_all do
    :inets.start
  end

  test "get request" do
    use_cassette "example_httpc_request" do
      {:ok, result} = :httpc.request('http://example.com')
      {{_http_version, _status_code = 200, _reason_phrase}, _headers, body} = result
      assert to_string(body) =~ ~r/Example Domain/
    end
  end
```

#### Custom Cassettes
You can manually define custom cassette json file for more flexible response control rather than just recoding the actual server response.
- Optional 2nd parameter of `ExVCR.Config.cassette_library_dir` method specifies the custom cassette directory. The directory is separated from vcr cassette one for avoiding mistakenly overwriting.
- Adding `custom: true` option to `use_cassette` macro indicates to use the custom cassette, and it just returns the pre-defined json response, instead of requesting to server.


```Elixir
defmodule ExVCR.MockTest do
  use ExUnit.Case
  import ExVCR.Mock

  setup_all do
    ExVCR.Config.cassette_library_dir("fixture/vcr_cassettes", "fixture/custom_cassettes")
    :ok
  end

  test "custom with valid response" do
    use_cassette "response_mocking", custom: true do
      assert HTTPotion.get("http://example.com", []).body =~ ~r/Custom Response/
    end
  end
```

The custom json file format is the same as vcr cassettes.

**fixture/custom_cassettes/response_mocking.json**
```javascript
[
  {
    "request": {
      "url": "http://example.com"
    },
    "response": {
      "status_code": 200,
      "headers": {
        "Content-Type": "text/html"
      },
      "body": "<h1>Custom Response</h1>"
    }
  }
]
```

### Recording VCR Cassettes
#### Matching
ExVCR uses url parameter to match request and cassettes. The "url" parameter in the json file is taken as regexp string.

#### Removing Sensitive Data
`ExVCR.Config.filter_sensitive_data(pattern, placeholder)` method can be used to remove sensitive data. It searches for string matches with `pattern`, which is a string representing a regular expression, and replaces with `placeholder`. Replacements happen both in URLs and request and response bodies.

```elixir
test "replace sensitive data" do
  ExVCR.Config.filter_sensitive_data("<PASSWORD>.+</PASSWORD>", "PLACEHOLDER")
  use_cassette "sensitive_data" do
    assert HTTPotion.get("http://something.example.com", []).body =~ ~r/PLACEHOLDER/
  end
end
```

`ExVCR.Config.filter_request_headers(header)` method can be used to remove sensitive data in the request headers. It checks if the `header` is found in the request headers and blanks out it's value with `***`.
```elixir
  test "replace sensitive data in request header" do
    ExVCR.Config.filter_request_headers("X-My-Secret-Token")
    use_cassette "sensitive_data_in_request_header" do
      body = HTTPoison.get!("http://localhost:34000/server?", ["X-My-Secret-Token": "my-secret-token"]).body
      assert body == "test_response"
    end

    # The recorded cassette should contain replaced data.
    cassette = File.read!("#{@dummy_cassette_dir}/sensitive_data_in_request_header.json")
    assert cassette =~ "\"X-My-Secret-Token\": \"***\""
    refute cassette =~  "\"X-My-Secret-Token\": \"my-secret-token\""

    ExVCR.Config.filter_request_headers(nil)
  end
```

#### Ignoring query params in url
If `ExVCR.Config.filter_url_params(true)` is specified, query params in url will be ignored when recording cassettes.

```elixir
test "filter url param flag removes url params when recording cassettes" do
  ExVCR.Config.filter_url_params(true)
  use_cassette "example_ignore_url_params" do
    assert HTTPotion.get(
      "http://localhost:34000/server?should_not_be_contained", []).body =~ ~r/test_response/
  end
  json = File.read!("#{__DIR__}/../#{@dummy_cassette_dir}/example_ignore_url_params.json")
  refute String.contains?(json, "should_not_be_contained")
```

#### Removing headers from response
If `ExVCR.Config.response_headers_blacklist(headers_blacklist)` is specified, the headers in the list will be removed from the response.

```elixir
  test "remove blacklisted headers" do
    use_cassette "original_headers" do
      assert Map.has_key?(HTTPoison.get!(@url, []).headers, "connection") == true
    end

    ExVCR.Config.response_headers_blacklist(["Connection"])
    use_cassette "remove_blacklisted_headers" do
      assert Map.has_key?(HTTPoison.get!(@url, []).headers, "connection") == false
    end

    ExVCR.Config.response_headers_blacklist([])
  end
```

#### Clearing Mock After Each Cassette
By default, mocking through `:meck.expect` is not cleared after each `use_cassette`. It can cause error when mixing actual/mocking accesses. In order to clear mock, please specify `[clear_mock: :true]` option through either of the followings.

```elixir
# For applying all the tests under the module.
defmodule ExVCR.Adapter.OptionsTest do
  use ExVCR.Mock, options: [clear_mock: true]
  use ExUnit.Case, async: false
...
```

```elixir
# For applying specific test.
use_cassette "option_clean_each", clear_mock: true do
  assert HTTPotion.get(@url, []).body == "test_response1"
end
```

#### Matching Options
##### matching against query params
By default, query params are not used for matching. In order to include query params, specify `match_requests_on: [:query]` for `use_cassette` call.

```elixir
test "matching query params with match_requests_on params" do
  use_cassette "different_query_params", match_requests_on: [:query] do
    assert HTTPotion.get("http://localhost/server?p=3", []).body =~ ~r/test_response3/
    assert HTTPotion.get("http://localhost/server?p=4", []).body =~ ~r/test_response4/
  end
end
```

##### matching against request body
By default, request body is not used for matching. In order to include query params, specify `match_requests_on: [:request_body]` for `use_cassette` call.

```elixir
test "matching query params with match_requests_on params" do
  use_cassette "different_request_body_params", match_requests_on: [:request_body] do
    assert HTTPotion.post("http://localhost/server", [body: "p=3"]).body =~ ~r/test_response3/
    assert HTTPotion.post("http://localhost/server", [body: "p=4"]).body =~ ~r/test_response4/
  end
end
```

### Default Configs
Default parameters for `ExVCR.Config` module can be specified in `config\config.exs` as follows.

```elixir
use Mix.Config

config :exvcr, [
  vcr_cassette_library_dir: "fixture/vcr_cassettes",
  custom_cassette_library_dir: "fixture/custom_cassettes",
  filter_sensitive_data: [
    [pattern: "<PASSWORD>.+</PASSWORD>", placeholder: "PASSWORD_PLACEHOLDER"]
  ],
  filter_url_params: false,
  response_headers_blacklist: []
]
```

If `exvcr` is defined as test-only dependency, describe the above statement in test-only config file (ex. `config\test.exs`) or make it conditional (ex. wrap with `if Mix.env == :test`).

### Mix Tasks
The following tasks are added by including exvcr package.
- [mix vcr](#mix-vcr-show-cassettes)
- [mix vcr.delete](#mix-vcrdelete-delete-cassettes)
- [mix vcr.check](#mix-vcrcheck-check-cassettes)
- [mix vcr.show](#mix-vcrshow-show-cassettes)
- [mix vcr --help](#mix-vcr-help-help)

#### [mix vcr] Show cassettes
```Shell
$ mix vcr
Showing list of cassettes in [fixture/vcr_cassettes]
  [File Name]                              [Last Update]
  example_httpotion.json                   2013/11/07 23:24:49
  example_ibrowse.json                     2013/11/07 23:24:49
  example_ibrowse_multiple.json            2013/11/07 23:24:48
  httpotion_delete.json                    2013/11/07 23:24:47
  httpotion_patch.json                     2013/11/07 23:24:50
  httpotion_post.json                      2013/11/07 23:24:51
  httpotion_put.json                       2013/11/07 23:24:52

Showing list of cassettes in [fixture/custom_cassettes]
  [File Name]                              [Last Update]
  method_mocking.json                      2013/10/06 22:05:38
  response_mocking.json                    2013/09/29 17:23:38
  response_mocking_regex.json              2013/10/06 18:13:45
```

#### [mix vcr.delete] Delete cassettes
The `mix vcr.delete` task deletes the cassettes that contains the specified pattern in the file name.
```Shell
$ mix vcr.delete ibrowse
Deleted example_ibrowse.json.
Deleted example_ibrowse_multiple.json.
```

If -i (--interactive) option is specified, it asks for confirmation before deleting each file.
```Shell
$ mix vcr.delete ibrowse -i
delete example_ibrowse.json? y
Deleted example_ibrowse.json.
delete example_ibrowse_multiple.json? y
Deleted example_ibrowse_multiple.json.
```

If -a (--all) option is specified, all the cassetes in the specified folder becomes the target for delete.

#### [mix vcr.check] Check cassettes
The `mix vcr.check` shows how many times each cassette is applied while executing `mix test` tasks. It is intended for verifying  the cassettes are properly used. `[Cassette Counts]` indicates the count that the pre-recorded json cassettes are applied. `[Server Counts]` indicates the count that server access is performed.

```Shell
$ mix vcr.check
...............................
31 tests, 0 failures
Showing hit counts of cassettes in [fixture/vcr_cassettes]
  [File Name]                              [Cassette Counts]    [Server Counts]
  example_httpotion.json                   1                    0
  example_ibrowse.json                     1                    0
  example_ibrowse_multiple.json            2                    0
  httpotion_delete.json                    1                    0
  httpotion_patch.json                     1                    0
  httpotion_post.json                      1                    0
  httpotion_put.json                       1                    0
  sensitive_data.json                      0                    2
  server1.json                             0                    2
  server2.json                             2                    2

Showing hit counts of cassettes in [fixture/custom_cassettes]
  [File Name]                              [Cassette Counts]    [Server Counts]
  method_mocking.json                      1                    0
  response_mocking.json                    1                    0
  response_mocking_regex.json              1                    0
```

The target test file can be limited by specifying test files, as similar as `mix test` tasks.

```Shell
$ mix vcr.check test/exvcr_test.exs
.............
13 tests, 0 failures
Showing hit counts of cassettes in [fixture/vcr_cassettes]
  [File Name]                              [Cassette Counts]    [Server Counts]
  example_httpotion.json                   1                    0
...
...
```

#### [mix vcr.show] Show cassettes
The `mix vcr.show` task displays the contents of cassettes json file in the prettified format.

```Shell
$ mix vcr.show fixture/vcr_cassettes/httpoison_get.json
[
  {
    "request": {
      "url": "http://example.com",
      "headers": [],
      "method": "get",
      "body": "",
      "options": []
    },
...
```

#### [mix vcr --help] Help
Displays helps for mix sub-tasks.

```Shell
$ mix vcr --help
Usage: mix vcr [options]
  Used to display the list of cassettes

  -h (--help)         Show helps for vcr mix tasks
  -d (--dir)          Specify vcr cassettes directory
  -c (--custom)       Specify custom cassettes directory

Usage: mix vcr.delete [options] [cassete-file-names]
  Used to delete cassettes

  -d (--dir)          Specify vcr cassettes directory
  -c (--custom)       Specify custom cassettes directory
  -i (--interactive)  Request confirmation before attempting to delete
  -a (--all)          Delete all the files by ignoring specified [filenames]

Usage: mix vcr.check [options] [test-files]
  Used to check cassette use on test execution

  -d (--dir)          Specify vcr cassettes directory
  -c (--custom)       Specify custom cassettes directory

Usage: mix vcr.show [cassete-file-names]
  Used to show cassette contents

```


##### Notes
If the cassette save directory is changed from the default, [-d, --dir] option (for vcr cassettes) and [-c, --custom] option (for custom cassettes) can be used to specify the directory.

### IEx Helper
`ExVCR.IEx` module provides simple helper functions to display the http request/response in json format, instead of recording in the cassette files.

```elixir
% iex -S mix
Erlang R16B03 (erts-5.10.4) ...
Interactive Elixir (0.12.5) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> require ExVCR.IEx
nil
iex(2)> ExVCR.IEx.print do
...(2)>   :ibrowse.send_req('http://example.com', [], :get)
...(2)> end
[
  {
    "request": {
      "url": "http://example.com",
      "headers": [],
      "method": "get",
      "body": "",
      "options": []
    },
    "response": {
      "type": "ok",
      "status_code": 200,
...
```

The adapter option can be specified as `adapter` argument of print function, as follows.

```elixir
% iex -S mix
Erlang R16B03 (erts-5.10.4) ...

Interactive Elixir (0.12.5) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> require ExVCR.IEx
nil
iex(2)> ExVCR.IEx.print(adapter: ExVCR.Adapter.Hackney) do
...(2)>   HTTPoison.get!("http://example.com").body
...(2)> end
[
  {
    "request": {
      "url": "http://example.com",
...
```

### Stubbing Response
Specifing `:stub` as fixture name allows directly stubbing the response header/body information based on parameter.

```Elixir
test "stub request works for HTTPotion" do
  use_cassette :stub, [url: "http://example.com", body: "Stub Response", status_code: 200] do
    response = HTTPotion.get("http://example.com", [])
    assert response.body =~ ~r/Stub Response/
    assert response.headers[:"Content-Type"] == "text/html"
    assert response.status_code == 200
  end
end

test "stub request works for HTTPoison" do
  use_cassette :stub, [url: "http://www.example.com", body: "Stub Response"] do
    response = HTTPoison.get!("http://www.example.com")
    assert response.body =~ ~r/Stub Response/
    assert response.headers["Content-Type"] == "text/html"
    assert response.status_code == 200
  end
end

test "stub request works for httpc" do
  use_cassette :stub, [url: "http://www.example.com",
                       method: "get",
                       status_code: ["HTTP/1.1", 200, "OK"],
                       body: "success!"] do

  {:ok, result} = :httpc.request('http://example.com')
  {{_http_version, _status_code = 200, _reason_phrase}, _headers, body} = result
  assert to_string(body) == "success!"
end
```

If the specified `:url` parameter doesn't match requests called inside the `use_cassette` block, it raises `ExVCR.InvalidRequestError`.

The `:url` can be regular expression string. Please note that you should use the `~r` sigil with `/` as delimiters.

```Elixir
test "match URL with regular expression" do
  use_cassette :stub, [url: "~r/(foo|bar)/", body: "Stub Response", status_code: 200] do
    # ...
  end
end

test "make sure to properly escape the /" do
  # escape /path/to/file
  use_cassette :stub, [url: "~r/\/path\/to\/file", body: "Stub Response", status_code: 200] do
    # ...
  end
end

test "the sigil delimiter cannot be anything else" do
  use_cassette :stub, [url: "~r{this-delimiter-doesn-not-work}", body: "Stub Response", status_code: 200] do
    # ...
  end
end
```

### TODO
- Improve performance, as it's very slow.
- Fix unstable behavior without `async: false`.
