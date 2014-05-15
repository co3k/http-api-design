# HTTP API 設計ガイド

## はじめに

このガイドは HTTP+JSON API の設計に関する慣習の数々を記述したものです。 [Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference) における成果物が基になっています。

このガイドは API (訳註: Heroku Platform API) に対して影響を与えたり、Heroku 内の新しい API 群の手引きとなります。また、私たちは、 Heroku 外の API 設計者の興味を惹くことも望んでいます。

設計に関する無駄な議論 (bikeshedding) を避け、一貫性やビジネスロジックにフォーカスをすることを目的としています。 _唯一、または理想的な方法_ に限らず、 API に関する _優れた、一貫した、 well-documented な方法_ を求めています。

本ドキュメントでは、基本的な HTTP+JSON API に精通している読者を想定としているため、それらの基礎事項については網羅していません。

このガイドに対する [貢献](CONTRIBUTING.md) も歓迎します。

## 目次

*  [適切なステータスコードを返しましょう](#return-appropriate-status-codes)
*  [可能な限り完全なリソースを提供しましょう](#provide-full-resources-where-available)
*  [リクエストボディでは JSON を受け付けましょう](#accept-serialized-json-in-request-bodies)
*  [リソースの (UU)ID を提供しましょう](#provide-resource-uuids)
*  [標準的なタイムスタンプを提供しましょう](#provide-standard-timestamps)
*  [ISO8601 形式の UTC 時間を使いましょう](#use-utc-times-formatted-in-ISO8601)
*  [一貫したパス形式を使いましょう](#use-consistent-path-formats)
*  [パスや属性は小文字にしましょう](#downcase-paths-and-attributes)
*  [外部キーのリレーションはネストしましょう](#nest-foreign-key-relations)
*  [利便性のために ID によらない参照も用意しましょう](#support-non-id-dereferencing-for-convenience)
*  [構造化されたエラーを生成しましょう](#generate-structured-errors)
*  [ETag によるキャッシュをサポートしましょう](#support-caching-with-etags)
*  [Request-Id でリクエストを追跡しましょう](#trace-requests-with-request-ids)
*  [範囲指定によるページングをしましょう](#paginate-with-ranges)
*  [rate limit のステータスを明示しましょう](#show-rate-limit-status)
*  [バージョンは Accepts ヘッダに含めましょう](#version-with-accepts-header)
*  [machine-readable な JSON schema を提供しましょう](#provide-machine-readable-json-schema)
*  [human-readable なドキュメントを提供しましょう](#provide-human-readable-docs)
*  [実行可能な例を提供しましょう](#provide-executable-examples)
*  [安定性 (stability) について記述しましょう](#describe-stability)
*  [SSL を必須としましょう](#require-ssl)
*  [デフォルトで読みやすい JSON を出力しましょう](#pretty-print-json-by-default)

### 適切なステータスコードを返しましょう

各レスポンスにて適切な HTTP ステータスコードを返しましょう。成功時レスポンスは本ガイドに沿って決められるべきです:

* `200`: `GET` リクエストが成功したか、 `DELETE` もしくは `PATCH` リクエストによる同期が完了した
* `201`: `POST` リクエストが成功し、同期が完了した
* `202`: 非同期の `POST` 、 `DELETE` 、 `PATCH` リクエストが完了する
* `206`: `GET` リクエストが成功したが、一部のレスポンスしか返さない (※ [範囲指定によるページングをしましょう](#paginate-with-ranges) を参照のこと)

ユーザーエラーやサーバーエラーにおけるステータスコードの手引きとしては [HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) を参照してください。

### 可能な限り完全なリソースを提供しましょう

そのレスポンスにおいて可能な限りは常に、完全なリソース表現 (例: すべての属性を伴うオブジェクト) を提供しましょう。 `PUT` 、 `PATCH` 、 `DELETE` の場合も含め、 200 もしくは 201 レスポンスにおいては完全なリソース表現を必ず提供しましょう。

例:

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 や 204 レスポンスでは完全なリソース表現は含まれません。

例:

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### リクエストボディでは JSON を受け付けましょう

`PUT` / `PATCH` / `POST` のリクエストボディでは form-encoded データではなく (あるいはそれに加えて) JSON を受け付けるようにしましょう。これによって JSON シリアライズされたレスポンスとの対称がとれます。

例:

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

### リソースの (UU)ID を提供しましょう

各リソースでは `id` 属性をデフォルトで用意しましょう。何か特別な理由がない限りは UUID を使いましょう。特に auto-incrementing な ID のような、そのサービス内のインスタンス間やリソース間でユニークにグローバルでアクセスできない ID は使わないでください。

UUID は小文字で `8-4-4-4-12` の形式で出力しましょう。

例:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### 標準的なタイムスタンプを提供しましょう

リソースに対して created_at や updated_at といったタイムスタンプを標準で提供しましょう。

例:

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

これらのタイムスタンプが意味をなさないようなリソースにおいては省略可能とします。

### ISO8601 形式の UTC 時間を使いましょう

入力においても出力においても、時間は UTC のみを受け付けます。 ISO8601 形式で表現しましょう。

例:

```
"finished_at": "2012-01-01T12:00:00Z"
```

### 一貫したパス形式を使いましょう

個々のリソースに対して特別な action を必要としないのが好ましいエンドポイントのレイアウトです。特別な action が必要な場合は、通例として `actions` という prefix の下に配置することでその旨を明確にしましょう。

```
/resources/:resource/actions/:action
```

例:

```
/runs/{run_id}/actions/stop
```

### パスや属性は小文字にしましょう

ホスト名にあわせて、小文字でダッシュ区切りのパス名を用いましょう。

例:

```
service-api.com/users
service-api.com/app-setups
```

属性も小文字ですが、 valid な JSON のキーとするために、属性名はアンダースコア区切りとしましょう。

例:

```
"service_class": "first"
```

### 外部キーのリレーションはネストしましょう

外部キー参照はネストしたオブジェクトとしてシリアライズしましょう。

例:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

良くない例:

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

このアプローチによって、レスポンスの構造の変更やトップレベルのレスポンスフィールドの導入をすることなく、関連したリソースに関する多くの情報を含むことができるようになります。

例:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

### 利便性のために ID によらない参照も用意しましょう

リソースの識別のために ID を提供することが、エンドユーザーにとっては面倒な場合があるかもしれません。たとえば、ユーザーは Heroku のアプリ名を期待しているけれども、実際にはアプリは UUID によって識別されているような場合です。このような場合は ID と名前の両方を受け付けたくなるでしょう。

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

名前だけを受け付けて ID は除外するというのはやめましょう。

### 構造化されたエラーを生成しましょう

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Document your error format and the possible error `id`s that clients may
encounter.

### ETag によるキャッシュをサポートしましょう

Include an `ETag` header in all responses, identifying the specific
version of the returned resource. The user should be able to check for
staleness in their subsequent requests by supplying the value in the
`If-None-Match` header.

### Request-Id でリクエストを追跡しましょう

Include a `Request-Id` header in each API response, populated with a
UUID value. If both the server and client log these values, it will be
helpful for tracing and debugging requests.

### 範囲指定によるページングをしましょう

Paginate any responses that are liable to produce large amounts of data.
Use `Content-Range` headers to convey pagination requests. Follow the
example of the [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
for the details of request and response headers, status codes, limits,
ordering, and page-walking.

### rate limit のステータスを明示しましょう

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.

### バージョンは Accepts ヘッダに含めましょう

Version the API from the start. Use the `Accepts` header to communicate
the version, along with a custom content type, e.g.:

```
Accept: application/vnd.heroku+json; version=3
```

Prefer not to have a default version, instead requiring clients to
explicitly peg their usage to a specific version.

### Minimize path nesting

In data models with nested parent/child resource relationships, paths
may become deeply nested, e.g.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limit nesting depth by preferring to locate resources at the root
path. Use nesting to indicate scoped collections. For example, for the
case above where a dyno belongs to an app belongs to an org:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### machine-readable な JSON schema を提供しましょう

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.

### human-readable なドキュメントを提供しましょう

Provide human-readable documentation that client developers can use to
understand your API.

If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.

In addition to endpoint details, provide an API overview with
information about:

* Authentication, including acquiring and using authentication tokens.
* API stability and versioning, including how to select the desired API
  version.
* Common request and response headers.
* Error serialization format.
* Examples of using the API with clients in different languages.

### 実行可能な例を提供しましょう

Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.

### 安定性 (stability) について記述しましょう

Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.

See the [Heroku API compatabilty policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.

Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.

### SSL を必須としましょう

Require SSL to access the API, without exception. It’s not worth trying
to figure out or explain when it is OK to use SSL and when it’s not.
Just require SSL for everything.

### デフォルトで読みやすい JSON を出力しましょう

The first time a user sees your API is likely to be at the command line,
using curl. It’s much easier to understand API responses at the
command-line if they are pretty-printed. For the convenience of these
developers, pretty-print JSON responses, e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Instead of e.g.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Be sure to include a trailing newline so that the user’s terminal prompt
isn’t obstructed.

For most APIs it will be fine performance-wise to pretty-print responses
all the time. You may consider for performance-sensitive APIs not
pretty-printing certain endpoints (e.g. very high traffic ones) or not
doing it for certain clients (e.g. ones known to be used by headless
programs).
