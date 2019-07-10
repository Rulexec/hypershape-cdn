hypershape-cdn
==============

It is files replication cluster.

Manages full copy of the files that are uploaded to any server of the cluster.

# API

All methods require authentication query-params:

`app=<app>&appToken=<token>`

Response object can contain `error` property, like:

```
{ "error": "500", "reason": "no_space" }
```

All methods are **async**. That means, they can respond before changed will be applied to whole cluster.

For example, `/static/delete` responds immediately, so files can be accessible even after receiving response.

## Static files

Static files are available at `/<app>/s/<filename>` (however recommended to serve them by `nginx` directly from static dir).

### `POST /_manage/static/put?name=<filename>`

Request:

```
<filecontent>
```

Response:

```
{}
```

### `POST /_manage/static/exists

Request:

```
{
	"files": ['path/to/file.ext', 'path/to/another-file.ext'],
	"type": "cluster",
	"timeout": 10,
	"notExists": false
}
```

Response:

```
{ "files": "10" }
```

Checks existance of files, responds with string of `0` or `1` (or `?` for cluster type) at the indexes of request files array.

`type` is optional, can be `"server" | "cluster"`. Checks on this server or on whole cluster.
If `"type": "cluster"`, result can be `?`, that means partial availability (some servers has file, some has not).

`timeout` is optional, if specified — waits `timeout` seconds if not all files exists.
It is useful to wait when bunch of files will be successfully replicated to whole cluster.

If `timeout` specified, `notExists` can be specified. If it is `true` — response will be delayed until all files will not exist.

### `POST /_manage/static/delete`

Request:

```
{ "files": ["path/to/file.ext", "path/to/another-file.ext"] }
```

Response:

```
{
	"indexed": [["path/to/file.ext", ["indexA/2", "indexB/5"]]],
}
```

Deletes files. Ignores non-existent.

If response contains `indexed`, it means that file was not deleted since it is belongs to single or multiple **files indexes**.
Files that are belong to **index** must be deleted by **index** deletion.

### `POST /_manage/static/list`

Request:

```
{
	"type": "all" | "withoutIndex",

	"cursor": "d0877042",
	"limit": 1000,
}
```

Possible responses:

```
{
	"cursor": "9f8ee23c",
	"files": ["path/to/file.ext", "path/to/another-file.ext"],
}
```

```
{
	"error": "cursor"
}
```

If server responds with `cursor` param — it means that more data available and you need to call `/_manage/static/list` again with that `cursor` param (without other arguments except optional `limit`).

Cursors can expire after some time if not requested. Also, if second cursor was requested — first will be disposed.
For example, you are performing request that responds with `"cursor":"A"` (that later responds with `"cursor":"B"`). You can do requests with `"cursor":"A"` as many times as you want, but after you will do request with `"cursor":"B"`, `A` cursor will be disposed (since server knows, that you successfully received data owned by `A` cursor.

**WARNING:** it lists only files that are available on the current server.

## Files index

Sometimes is useful to provide list of paths to bunch of files.
For example, to frontend application entry points (that later can be used by some server/script to apply deployment).

Also, file indexes are maintain real existance of files (it is impossible to get file index that point to non-existing files).

There is two phases:

* **create** — tells to server, that you want to create files index, where you list files (that can be not yet present) that are you want to include in this index.
* **apply** — asks cluster to synchronically apply files index. It will check availability of all files and lock them from deletion by `/static/` API.

Once index was applied — there is no possibility to remove files that are belong to it. You need to delete all indexes, that contain file that you want to remove.

Example:

```
GET /<app>/i/<indexName>

=====

{
	"version": 6,
	"entry.js": "/<app>/s/bundle/entry_80043102e5bb3553.js",
	"entry.css": "/<app>/s/bundle/entry_65fcab2ebcaf6bef.css",
	"manifest.js": "/<app>/s/bundle/manifest_9af47d2689071f57.js"
}
```

### `POST /_manage/index/create`

Request:

```
{
	"name": "<indexName>",
	"files": {
		"entry.js": "bundle/entry_731bf479d791aae9.js",
		"entry.css": "bundle/entry_76f73a56ce45bfe.css",
		"manifest.js": "bundle/manifest_a7c17f74f90868b.js"
	}
}
```

Response:

```
{
	"indexVersion": 7,
	"actualVersion": 6,
	"ready": false
}
```

Creates file index (but not applies).

If `ready` is `true` then all specified files are available on the whole cluster and file index can be applied.

### `POST /_manage/index/apply`

Request

```
{
	"name": "<indexName>",
	"indexVersion": 7,
	"fromVersion": 6
}
```

Possible responses:

```
{}
```

```
{
	"error": "not_ready",
	"missingFiles": [
		"bundle/entry_76f73a56ce45bfe.css"
	]
}
```

```
{
	"error": "race",
	"indexVersion": 7
}
```

Tries to apply new version files index. Requires `fromVersion` that will be checked against actual version.
If version was changed before your request — you will get error (since somebody are updated file index before you).

### `GET /_manage/index/info/<indexName>`

Optional arguments:

`fields` — comma-separated of interesting fields (like `indexVersion,applyTime` if you don't need files list).

`<indexName>` can contain version, like: `indexA/6`.

Response:

```
{
	"indexVersion": 6,
	"creationTime": 1562779060540,
	"applyTime": 1562779080400,
	"files": {
		"entry.js": "/<app>/s/bundle/entry_80043102e5bb3553.js",
		"entry.css": "/<app>/s/bundle/entry_65fcab2ebcaf6bef.css",
		"manifest.js": "/<app>/s/bundle/manifest_9af47d2689071f57.js"
	}
}
```

### `POST /_manage/index/delete/<indexName>`

Request:

```
{
	"withFiles": true,
	"wholeIndex": true
}
```

Possible responses:

```
{}
```

```
{
	"error": "whole_index",
	"reason": "specify wholeIndex:true"
}
```

`<indexName>` can contain version (`indexA/42`).

If no version specified — you need to pass `"wholeIndex":true` (it is like `rm -rf --no-preserve-root /`).

If `withFiles` specified — files that are belong to index will be removed (that are not belong to some other index).