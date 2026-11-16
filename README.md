# Rick and Morty API client for Dart

[![pub package](https://img.shields.io/pub/v/pauln10v_rick_and_morty_api.svg)](https://pub.dev/packages/pauln10v_rick_and_morty_api)

A small, typed Dart client for [The Rick and Morty API](https://rickandmortyapi.com).
It covers the three resources (characters, locations and episodes), supports
lookup by id, by a list of ids, filtering and pagination, and reports failures
as values instead of throwing.

This is an unofficial package and is not affiliated with the API's authors.

- [Installation](#installation)
- [Getting started](#getting-started)
- [Handling results](#handling-results)
- [Examples](#examples)
    - [Get by id](#get-by-id)
    - [Get by several ids](#get-by-several-ids)
    - [Get all](#get-all)
    - [Filter](#filter)
    - [Pagination](#pagination)
    - [Get endpoints](#get-endpoints)
- [Error handling](#error-handling)
- [Testing and mocking](#testing-and-mocking)
- [Support the API](#support-the-api)

## Installation

```sh
dart pub add pauln10v_rick_and_morty_api http
# or, in a Flutter project
flutter pub add pauln10v_rick_and_morty_api http
```

The client takes an `http.Client` from `package:http`, so your app decides how
requests are sent (and can share one client across the whole app).

## Getting started

```dart
import 'package:http/http.dart' as http;
import 'package:pauln10v_rick_and_morty_api/service/rickandmorty_api.dart';
import 'package:pauln10v_rick_and_morty_api/service/rickandmorty_rest_api.dart';

Future<void> main() async {
  final client = http.Client();
  final RickAndMortyApi api = RickAndMortyRestApi(
    client: client,
    baseUrl: 'https://rickandmortyapi.com',
  );

  final result = await api.getCharacterById(1);
  if (result.is1) {
    final rick = result.as1;
    print('${rick.name} (${rick.species})');
  }

  client.close();
}
```

Declare the variable as the `RickAndMortyApi` interface, as above, and depend on
the interface in the rest of your code. `RickAndMortyRestApi` is the default
implementation that talks to the real API.

## Handling results

Every method returns a `Future<ApiResult<T>>`. An `ApiResult<T>` is a union of
three possible outcomes, and exactly one of them is set:

| Variant | Check | Read | Meaning |
| ------- | ----- | ---- | ------- |
| 1 | `result.is1` | `result.as1` | Success. The parsed data of type `T`. |
| 2 | `result.is2` | `result.as2` | The server answered with a `ProblemDetails` body. |
| 3 | `result.is3` | `result.as3` | Any other failure, as an `ApiError`. |

```dart
final result = await api.getLocationById(1);

if (result.is1) {
  final earth = result.as1;
  print('${earth.name}: ${earth.type}, ${earth.dimension}');
} else if (result.is2) {
  print('Problem: ${result.as2}');
} else {
  print('Request failed: ${result.as3}');
}
```

Nothing is thrown for network errors, timeouts, unexpected status codes or
unparsable responses; they all come back as variant 3.

## Examples

All methods are asynchronous. The snippets below assume the `api` object and the
imports from [Getting started](#getting-started), plus the DTO imports shown in
each example.

### Get by id

```dart
final rick = await api.getCharacterById(1);
final earth = await api.getLocationById(1);
final pilot = await api.getEpisodeById(1);
```

### Get by several ids

The result is an `Iterable`. A single id works too, even though the API itself
answers with an object instead of an array in that case. An empty list returns an
empty result without calling the API.

```dart
final theSmiths = await api.getCharacterByIds([2, 3, 4, 5]);
final places = await api.getLocationByIds([1, 3]);
final seasonOne = await api.getEpisodeByIds(List.generate(11, (i) => i + 1));

if (theSmiths.is1) {
  for (final character in theSmiths.as1) {
    print(character.name);
  }
}
```

### Get all

The list methods return one page at a time, wrapped in a `PagedResponse` with
`info` (the API's paging details) and `results`.

```dart
final characters = await api.getCharacters();
final locations = await api.getLocations();
final episodes = await api.getEpisodes();

if (characters.is1) {
  final page = characters.as1;
  print('${page.results.length} characters on this page');
}
```

### Filter

Pass a filter object to narrow the results. See the
[API documentation](https://rickandmortyapi.com/documentation/#filter-characters)
for what each field matches.

```dart
import 'package:pauln10v_rick_and_morty_api/dto/character.dart';
import 'package:pauln10v_rick_and_morty_api/dto/episode.dart';
import 'package:pauln10v_rick_and_morty_api/dto/location.dart';

final aliveRicks = await api.getCharacters(
  filter: CharacterFilter(name: 'rick', status: CharacterStatus.alive),
);

final planets = await api.getLocations(
  filter: LocationFilter(type: 'planet'),
);

final seasonOne = await api.getEpisodes(
  filter: EpisodeFilter(episode: 'S01'),
);
```

A filter that matches nothing is reported by the API as a 404, which this client
returns as an `ApiResponseError` (see [Error handling](#error-handling)).

### Pagination

Each paged response carries the details needed to move through the pages. To open
a specific page, request its URL with the `...ByUri` methods. They also accept
filtered URLs.

```dart
final secondPage = await api.getCharactersByUri(
  Uri.parse('https://rickandmortyapi.com/api/character?page=2'),
);

final filteredPage = await api.getCharactersByUri(
  Uri.parse('https://rickandmortyapi.com/api/character?page=2&name=rick'),
);
```

`getLocationsByUri` and `getEpisodesByUri` work the same way.

### Get endpoints

`getEndpoints()` returns the URLs of the available resources. The client uses it
internally and caches a successful response, so it is only fetched once. Pass
`force: true` to bypass the cache, which also makes it a handy way to check that
the server is reachable.

```dart
final endpoints = await api.getEndpoints(force: true);
print(endpoints.is1 ? 'API is up' : 'API is unreachable');
```

## Error handling

Failures arrive as `ProblemDetails` (variant 2) or `ApiError` (variant 3).
The `ApiError` variants are:

| Type | When |
| ---- | ---- |
| `ApiResponseError` | The API answered with a non-success status, for example 404 when nothing matches. Exposes `statusCode` and the response `body`. |
| `CloudflareTooManyRequestsError` | The API answered with 429 because of rate limiting. |
| `ApiExceptionError` | The request failed before a usable response arrived: no connection, the 15-second timeout, or a response that could not be parsed. |

```dart
import 'package:pauln10v_rick_and_morty_api/errors/api_errors.dart';

final result = await api.getCharactersByUri(
  Uri.parse('https://rickandmortyapi.com/api/character?name=NEVER_EXISTED'),
);

if (result.is3) {
  final error = result.as3;
  if (error is ApiResponseError && error.statusCode == 404) {
    print('Nothing found');
  } else if (error is CloudflareTooManyRequestsError) {
    print('Slow down, rate limit reached');
  } else {
    print('Request failed: $error');
  }
}
```

## Testing and mocking

`RickAndMortyApi` is an interface, so you can replace it in your own tests
without any network access.

```dart
import 'package:mocktail/mocktail.dart';

class MockRickAndMortyApi extends Mock implements RickAndMortyApi {}
```

Alternatively, pass an `http.Client` from `package:http/testing.dart`
(`MockClient`) to `RickAndMortyRestApi` to test code that depends on the
concrete class.

## Support the API

The data is provided by The Rick and Morty API, which is run by volunteers. If
this package is useful to you, please consider
[helping to maintain its infrastructure](https://rickandmortyapi.com/help-us), and
see the [project page](https://rickandmortyapi.com/about) to learn more about it.