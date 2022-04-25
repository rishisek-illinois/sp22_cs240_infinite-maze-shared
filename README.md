# Infinite Maze API

## 0. Course-Wide Microservice

This microservice will be used as the middleware for the entire CS 240 course.  You are encouraged (and be eligible for EC) by contributing pull requests to this repository to improve the middleware.  Your MGs must work with this middleware.

### Attribution

- The basis of this work came from Peter's Week #1 submission.
- CS 240 TA, Patrick, transformed the proof-of-concept frontend to a paper.js rendering.
- The server page was contributed by "Team aaryab2-atanwar2"'s Week #1 submission.
- You will be adding the next big thing to this maze! :)


## 1. Introduction

This API is designed to work properly with the proof-of-concept front-end implementation, while being flexible enough to accommodate for future changes. It allows an indefinite number of back-end maze generators (MGs) to be registered and subsequently selected to provide the front-end with maze segments.

The API itself is a Flask application that keeps track of available MGs. MGs can be registered and unregistered at any point with a `PUT` request. When called, the API will randomly select a registered MG server and send a request for maze data. The data in the response is then forwarded to the front-end.

MG servers may specify their RNG weight when registering. The API will use these weights when selecting a random MG.

To allow for extensibility, the front-end may pass additional URL parameters to the API. These parameters are forwarded directly to the selected MG. MGs may also pass information back to the front-end by including it in their JSON responses.

### Setup and run

The included API and MG servers require Python 3. To **install required modules**, run:
```
pip install -r requirements.txt
```

To **start the API server**:

```
flask run
```
or

```
python3 -m flask run
```

## 2. Communication with maze generators

Once a generator is selected, the API sends a `GET` request to `<MG_url>/generate`. The maze generator (MG) server must return a JSON, which is forwarded to the front-end.

For example, if the API selects a generator added with URL `localhost:24000`, the API will send a `GET` request to `localhost:24000/generate`. If the MG server responds with a `200`-level response, the included JSON is sent in the response to the front-end.

The MG server's response must include the HTTP header:

```
Content-Type: application/json
```

The JSON sent by the MG server may have any form. With the current front-end implementation, it may look like:

```
{"geom": ["92c", "4a1", "386"]}
```

MG servers may include any additional keys and values in the JSON if needed for extra functionality.

The MG server must respond with an HTTP status code between `200` and `299` inclusive. If the response code is outside this range, an error will be sent to the front-end.

## 3. Communication with front-end

A `GET` request to `<API_URL>/generateSegment` will return a maze segment generated by a randomly selected maze generator (MG) server in JSON form. For example, to request a maze segment from an API running at `localhost:5000`:

```
curl -X GET localhost:5000/generateSegment
```

To specify a particular MG server, send a `GET` request to `<API_URL>/generateSegment/<MG_name>` where `<MG_name>` is the name given when the MG server was added (see [Adding maze generator servers](#adding-maze-generator-servers)). For example, to request the MG server `'generator1'`:

```
curl -X GET localhost:5000/generateSegment/generator1
```

If the requested MG server has not been added, the API will return a `404` response.

Additional URL parameters may be included for additional functionality. See [Passing parameters](#5-passing-parameters).

## 4. Adding MGs

The API maintains a list of available maze generator (MG) servers. Users may add and remove MG servers at any time, and may request a list of all available MG servers.

### Adding maze generator servers

To add an MG server, send a `PUT` request to `<API_URL>/addMG`. This request must include a JSON with keys `'name'`, `'url'`, and `'author'`. For example:

```
{ "name": "generator1",
  "url": "http://127.0.0.1:24000/",
  "author": "Your Name",
  "weight": 0.5 }
```

The `'name'` value will be needed to remove the MG server from the API's list of available servers.

An optional `'weight'` key may be included to increase or decrease the probability of being randomly selected. The value must be a number greater than 0. If omitted, the weight will default to 1.

If the `'name'` provided is already in use, the stored URL will be overwritten.

The API will respond with status code `400` if the JSON is malformed.


## 5. Passing parameters

`GET` requests to `<API_URL>/generateSegment` or `<API_URL>/generateSegment/<MG_name>` may include URL parameters to request additional functionality from maze generator (MG) servers. The API includes these URL parameters in its subsequent request to the selected MG server.

For example, if the API is running at `localhost:5000` and one wants to send URL parameters `random=true&size=7` directly to the MG added with name `generator1`, send a `GET` request to:

```
http://localhost:5000/generator1?random=true&size=7
```

Since MG servers are allowed to respond with JSON of any form, they have complete flexibility over any additional information to send to the front-end. Additional key-value pairs may be added besides the `'geom'` key required by the current front-end implementation.

For a proof-of-concept, see the [`static_given`](IMs.md#1-static_given) MG.

## 6. Global maze state

`GET` requests to `<API_URL>/generateSegment` can specify `row` and `col` URL parameters to retrieve data for a 7x7 maze segment in a certain position. If the segment has already been generated by a previous request to the same coordinates, the same segment will be returned without communicating with a maze generator (MG) server.

`row` and `col` will default to 0 if not specified. This feature does not work with requests to `<API_URL>/generateSegment/<mg_name>`.

To reset the global maze state, send a `DELETE` request to `<API_url>/resetMaze`.

## 7. Appendix

### HTTP status codes reference

| Route                          | Response code | Situation                                                      |
|------------------------------|---------------|----------------------------------------------------------------|
| (all)                        | 200           | Request was fulfilled normally.                                |
| (all)                        | 500           | An internal error occurred.                                    |
| `/generateSegment`           | 503           | No maze generators are available. Add more with [/addMG](#adding-maze-generator-servers). |
| `/generateSegment/<MG_name>` | 404           | Provided MG name has not been added.                           |
| `/addMG`                     | 400           | Provided JSON is malformed and missing a required key, or provides an invalid weight.         |
| `/resetMaze`                 | 304           | Maze was already empty.                                        |


### Tips and tricks

- Configure your MG server to add itself to the API on startup, as manually sending the required JSON can be tedious (see [Adding maze generator servers](#adding-maze-generator-servers)). Note that if the MG server has already been added, sending a second identical request to `/addMG` will have no effect.

___

# Implemented Advanced Features

## 1. Arbitrary communication

See [Passing parameters](#5-passing-parameters).

Front-end clients and back-end MG servers can send additional data directly to each other. The front-end can include additional URL parameters, and the back-end can include additional data in the returned JSON.

This allows for flexibility and extensibility with future features. Here are some ideas:

- The front-end can send data about the player's current position to adjust MG behavior.
- The front-end can send data about available space to allow for maze segments of varying dimensions.
- Back-end MGs can send additional data about maze segments and cells, such as cell colors and image backgrounds.
 - The front-end can send player-specific data through URL parameters to generate maze segments customized to individual players.

 ## 2. Weighted RNG

 When adding an MG to the API, you may specify a "weight" value (see [Adding maze generator servers](#adding-maze-generator-servers)). The RNG will use these values as weights when selecting a random MG server (after a request to `<API_URL>/generateSegment`).

The `'weight'` key is optional, and the default weight is `1`. Floating-point weight values like `0.1` are allowed, but weights must be greater than 0.
