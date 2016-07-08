The Basecamp 3 API
==================

Welcome to the Basecamp 3 API! Looking to integrate your service with Basecamp 3, or create your own app in concert with data inside of Basecamp 3? We're happy to have you!

If you have a specific feature request or if you found a bug, [please open a GitHub issue](https://github.com/basecamp/bc3-api/issues/new). We encourage forking these docs for local reference, and will happily accept pull request with improvements.

To talk with us and other developers about the API, [post a question on StackOverflow](http://stackoverflow.com/questions/ask) tagged `basecamp` or [open a support ticket](https://basecamp.com/support) if you need help from us directly.

**Please note:** During this early access period (while this repo is private), please hold on with announcing your API integration publicly. Once we're fully public you can go for it.

Compatibility with previous Basecamp APIs
-----------------------------------------

The Basecamp 3 API is not compatible with the [Basecamp Classic API](https://github.com/basecamp/basecamp-classic-api) or the [Basecamp 2 API](https://github.com/basecamp/bcx-api). All integrations will start fresh with the new API. The core ingredients are the same, though: this is a REST-style API that uses JSON for serialization and OAuth 2 for authentication.


What's different?
-----------------

If you've used a previous version of the Basecamp API, you will need to adapt your integration code. Here's a quick summary of changes:

- "Projects" are now Basecamps.
- We're requiring OAuth for [authentication](#authentication). No more Basic auth.
- All requests must end in `.json`
- Many IDs are numeric, but many are now [Signed Global IDs (SGIDs)](https://github.com/rails/globalid#signed-global-ids), also known as "Squids"
- [Pagination](#pagination) is now performed via the `Link` and `X-Total-Count` headers.


Making a request
----------------

All URLs start with **`https://3.basecampapi.com/999999999/`**. HTTPS only. The path is prefixed with the account ID, but no `/api/v1` API prefix. Also note the different domain!

To make a request for all the projects on your account, you'd append the projects index path to the base url to form something like https://3.basecampapi.com/999999999/projects.json. In cURL, that looks like:

``` shell
curl -H "Authorization: Bearer $ACCESS_TOKEN" -H 'User-Agent: MyApp (yourname@example.com)' https://3.basecampapi.com/999999999/projects.json
```

To create something, it's the same deal except you also have to include the `Content-Type` header and the JSON data:

``` shell
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'User-Agent: MyApp (yourname@example.com)' \
  -d '{ "name": "My new project!" }' \
  https://3.basecampapi.com/999999999/projects.json
```

That's all! Throughout this guide we've included "Copy as cURL" examples. If you'd like to try this out in your shell, copy your OAuth Access token into your clipboard and run:

``` shell
export ACCESS_TOKEN=PASTE_ACCESS_TOKEN_HERE
export ACCOUNT_ID=999999999
```

Then you should be able to easily copy + paste any example from our docs. After pasting a cURL example, you could pipe it to a JSON pretty printer to make it a little more readable. Try [jsonpp](https://jmhodges.github.io/jsonpp/) or `json_pp` on OSX:

``` shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" https://3.basecampapi.com/999999999/projects.json | json_pp
```

Authentication
--------------

If you're making a public integration with Basecamp for others to enjoy, you must use OAuth 2. This allows users to authorize your application to use Basecamp on their behalf without having to copy/paste API tokens or touch sensitive login info.

Read the [authentication guide](https://github.com/basecamp/api/blob/master/sections/authentication.md) to get started.


Identify your app
-----------------

You must include a `User-Agent` header with the name of your application and a link to it or your email address so we can get in touch in case you're doing something wrong (so we may warn you before you're blacklisted) or something awesome (so we may congratulate you). Here's a couple of examples:

    User-Agent: Freshbooks (http://freshbooks.com/contact.php)
    User-Agent: Fabian's Ingenious Integration (fabian@example.com)

If you don't supply this header, you will get a `400 Bad Request` response.


JSON only
---------

We use JSON for all API data. Style: no root element and snake\_case for object keys. This means that you have to send `Content-Type: application/json; charset=utf-8` when you're POSTing or PUTing data into Basecamp. All API URLs end in .json to indicate that they return JSON.

You'll receive a `415 Unsupported Media Type` response code if you attempt to use a different URL suffix or leave out the `Content-Type` header.


Pagination
----------

Most collection APIs paginate their results. The first request returns up to 50 records. The Basecamp 3 API follows the [RFC5988 convention](https://tools.ietf.org/html/rfc5988) of using the `Link` header to provide URLs for the `next` page. Follow this URLs to retrieve the next page of data, and please don't build the pagination URLs yourself! Here's an example from the [Messages API][2] for requesting the third page (of 50) when 300 are available:

```
<https://3.basecampapi.com/999999999/buckets/2085958496/messages.json?page=4>; rel="next"
```

Quick note: If the `Link` header is blank, and you have some results, then that's the only page of data! We also provide the `X-Total-Count` header, which displays the total amount of resources in the collection you are fetching.


Rich content
------------

Many resources accept HTML for content to display in Basecamp. If you're accepting content from a web browser, we'd highly encourage you to check out [Trix](https://github.com/basecamp/trix), our rich content editor. If you are sending HTML tags directly, here's what we allow:

* `div`
* `strong`
* `em`
* `strike`
* `a` (with `href` attribute)
* `pre`
* `ol`
* `ul`
* `li`
* `blockquote`
* `br`

For all fields that support rich content, our API provides two types of content to consume: one for rich content (for example, `description_html`), and one for plain text (`description_text`) that has any HTML tags removed. When sending data to our API, clients can only create and update resources with rich content (for example, the `description_html` field).

Please note: The formatting/DOM layout of our HTML is subject to change. If you require or are relying on a more consistent format in your API integration, we'd love to hear about how you're using it. Open up an issue or [email us](support+api@basecamp.com) with how you're using rich content and we'll be happy to take a look!


Use HTTP caching
----------------

You must make use of the HTTP freshness headers to speed up your app and lighten the load on our servers. Most API responses will include an `ETag` or `Last-Modified` header. When you first request a resource, store these values. On subsequent requests, submit them back to us as `If-None-Match` and `If-Modified-Since`, respectively. If the resource hasn't changed since your last request, you'll get a `304 Not Modified` response with no body, saving you the time and bandwidth of sending something you already have.


Handling errors
---------------

If Basecamp is having trouble, you might see a 5xx error. `500` means that the app is entirely down, but you might also see `502 Bad Gateway`, `503 Service Unavailable`, or `504 Gateway Timeout`. It's your responsibility in all of these cases to retry your request later.


Rate limiting
-------------

You can perform up to 50 requests per 10 second period from the same IP address for the same account. If you exceed this limit, you'll get a [429 Too Many Requests](http://tools.ietf.org/html/draft-nottingham-http-new-status-02#section-4) response for subsequent requests. Check the `Retry-After` header to see how many seconds to wait before retrying the request.

We recommend baking 429 response handling in to your HTTP handling at a low level so your integration gracefully and automatically handles retries.

API endpoints
-------------

| URL | Endpoint | Ready to use? |
| :--- | :--- | :---: |
| ▾ **[Basecamps](sections/basecamps.md#basecamps)** | | |
| [GET /projects.json](sections/basecamps.md#get-basecamps) | Get Basecamps | 👍 |
| [GET /projects/archive.json](sections/basecamps.md#get-archived-basecamps) | Get archived Basecamps | 👍 |
| [GET /projects/trash.json](sections/basecamps.md#get-trashed-basecamps) | Get trashed Basecamps | 👍 |
| [GET /projects/1.json](sections/basecamps.md#get-a-basecamp) | Get a Basecamp | 👍 |
| [POST /projects.json](sections/basecamps.md#create-a-basecamp) | Create a Basecamp | 👍 |
| [PUT /projects/1.json](sections/basecamps.md#update-a-basecamp) | Update a Basecamp | 👍 |
| [DELETE /projects/1.json](sections/basecamps.md#trash-a-basecamp) | Trash a Basecamp | 👍 |
| ▾ **[Campfires](sections/campfire.md#campfires)** | | |
| [GET /chats.json](sections/campfires.md#get-campfires) | Get Campfires | 👍 |
| [GET /buckets/1/chats/2.json](sections/campfires.md#get-a-campfire) | Get a Campfires | 👍 |
| [GET /buckets/1/chats/2/lines.json](sections/campfires.md#get-campfire-lines) | Get Campfire lines | 👍 |
| [GET /buckets/1/chats/2/lines/3.json](sections/campfires.md#get-a-campfire-line) | Get a Campfire line | 👍 |
| [POST /buckets/1/chats/2/lines.json](sections/campfires.md#create-a-campfire-line) | Create a Campfire line | 👍 |
| ▾ **[Client approvals](sections/client_approvals.md#client-approvals)** | | |
| [GET /buckets/1/client/approvals.json](sections/client_approvals.md#get-client-approvals) | Get client approvals | 👍 |
| [GET /buckets/1/client/approvals/2.json](sections/client_approvals.md#get-a-client-approvals) | Get a client approval | 👍 |
| ▾ **[Client correspondences](sections/client_correspondences.md#client-correspondences)** | | |
| [GET /buckets/1/client/correspondences.json](sections/client_correspondences.md#get-client-correspondences) | Get client correspondences | 👍 |
| [GET /buckets/1/client/correspondences/2.json](sections/client_correspondences.md#get-a-client-correspondences) | Get a client correspondence | 👍 |
| ▾ **[Client replies](sections/client_replies.md#client-replies)** | | |
| [GET /buckets/1/client/recordings/2/replies.json](sections/client_replies.md#get-client-replies) | Get client replies | 👍 |
| [GET /buckets/1/client/recordings/2/replies/3.json](sections/client_replies.md#get-a-client-reply) | Get a client reply | 👍 |
| ▾ **[Comments](sections/comments.md#comments)** | | |
| [GET /buckets/1/recordings/3/comments.json](sections/comments.md#get-comments) | Get comments | 👍 |
| [GET /buckets/1/comments/2.json](sections/comments.md#get-a-comment) | Get a comment | 👍 |
| [POST /buckets/1/recordings/3/comments.json](sections/comments.md#create-a-comment) | Create a comment | 👍 |
| [PUT /buckets/1/comments/2.json](sections/comments.md#update-a-comment) | Update a comment | 👍 |
| ▾ **[Messages](sections/messages.md#messages)** | | |
| [GET /buckets/1/messages/2.json](sections/messages.md#get-a-message) | Get a message | 👍 |
| [POST /buckets/1/message_boards/3/messages.json](sections/messages.md#create-a-message) | Create a message | 👍 |
| [PUT /buckets/1/messages/2.json](sections/messages.md#update-a-message) | Update a message | 👍 |
| ▾ **[Message Boards](sections/message_boards.md#message-boards)** | | |
| [GET /buckets/1/message_boards/2.json](sections/message_boards.md#get-message-board) | Get message board | 👍|
| [GET /buckets/1/message_boards/3/messages.json](sections/messages.md#get-messages) | Get messages | 👍|
| ▾ **[People](sections/people.md#people)** | | |
| [GET /people.json](sections/people.md#get-all-people) | Get all people | 👍|
| [GET /projects/1/people.json](sections/people.md#get-people-on-a-basecamp) | Get people on a Basecamp | 👍|
| [PUT /projects/1/people/users.json](sections/people.md#update-who-can-access-a-basecamp) | Update who can access a Basecamp | 👍 |
| [GET /circles/people.json](sections/people.md#get-pingable-people) | Get pingable people | 👍|
| [GET /people/2.json](sections/people.md#get-person) | Get person | 👍|
| [GET /my/profile.json](sections/people.md#get-my-personal-info) | Get my personal info | 👍|
| ▾ **[Questionnaires](sections/questionnaires.md#questionnaires)** | | |
| [GET /buckets/1/questionnaires/2.json](sections/questionnaires.md#get-a-questionnaire) | Get questionnaire | 👍|
| ▾ **[Questions](sections/questions.md#questions)** | | |
| [GET /buckets/1/questionnaires/2/questions.json](sections/questions.md#get-questions) | Get questions | 👍 |
| [GET /buckets/1/questions/2.json](sections/questions.md#get-a-question) | Get a question | 👍 |
| ▾ **[Question answers](sections/question_answers.md#question-answers)** | | |
| [GET /buckets/1/questions/2/answers.json](sections/question_answers.md#get-question-answers) | Get question answers | 👍 |
| [GET /buckets/1/question_answers/2.json](sections/question_answers.md#get-a=question-answer) | Get a question answer | 👍 |
| ▾ **[Recordings](sections/recordings.md#recordings)** | | |
| [PUT /buckets/1/recordings/2/status/trashed.json](sections/recordings.md#trash-a-recording) | Trash a recording | 👍|
| ▾ **[Schedules](sections/schedules.md#schedules)** | | |
| [GET /buckets/1/schedules/2.json](sections/schedules.md#get-schedule) | Get a schedule | 👍 |
| ▾ **[Schedule Entries](schedule_entries.md#schedule-entries)** | | |
| [GET /buckets/1/schedules/3/entries.json](sections/schedule_entries.md#get-schedule-entries) | Get schedule entries | 👍 |
| [GET /buckets/1/schedule_entries/2.json](sections/schedule_entries.md#get-a-schedule-entry) | Get a schedule entry | 👍 |
| [POST /buckets/1/schedules/3/entries.json](sections/schedule_entries.md#create-a-schedule-entry) | Create a schedule entry | 👍 |
| [PUT /buckets/1/schedule_entries/2.json](sections/schedule_entries.md#update-a-schedule-entry) | Update a schedule entry | 👍 |
| ▾ **[To-do sets](sections/todosets.md#to-do-sets)** | | |
| [GET /buckets/1/todosets/2.json](sections/todosets.md#get-to-do-set) | Get to-do set | 👍|
| ▾ **[To-do lists](sections/todolists.md#to-do-lists)** | | |
| [GET /buckets/1/todolists/2.json](sections/todolists.md#get-a-to-do-list) | Get a to-do list | 👍|
| [POST /buckets/1/todosets/3/todolists.json](sections/todolists.md#create-a-to-do-list) | Create a to-do list | 👍|
| [PUT /buckets/1/todolists/2.json](sections/todolists.md#update-a-to-do-list) | Update a to-do list | 👍|
| ▾ **[To-dos](sections/todos.md#to-dos)** | | |
| [GET /buckets/1/todolists/3/todos.json](sections/todos.md#get-to-dos) | Get to-dos | 👍|
| [GET /buckets/1/todos/2.json](sections/todos.md#get-a-to-do) | Get a to-do | 👍|
| [POST /buckets/1/todolists/3/todos.json](sections/todos.md#create-a-to-do) | Create a to-do | 👍|
| [PUT /buckets/1/todos/2.json](sections/todos.md#update-a-to-do) | Update a to-do | 👍|
| [POST /buckets/1/todos/2/completion.json](sections/todos.md#complete-a-to-do) | Complete a to-do | 👍|
| [DELETE /buckets/1/todos/2/completion.json](sections/todos.md#uncomplete-a-to-do) | Uncomplete a to-do | 👍|
| [PUT /buckets/1/todos/2/position.json](sections/todos.md#reposition-a-to-do) | Reposition a to-do | 👍|
| ▾ **[Vaults](sections/vaults.md#vaults)** | | |
| [GET /buckets/1/vaults/2/vaults.json](sections/vaults.md#get-vaults) | Get vaults | 👍|
| [GET /buckets/1/vaults/2.json](sections/vaults.md#get-a-vault) | Get a vault | 👍|
| [POST /buckets/1/vaults/2/vaults.json](sections/vaults.md#create-a-vault) | Create a vault | 👍|
| [PUT /buckets/1/vaults/3.json](sections/vaults.md#update-a-vault) | Update a vault | 👍|
| **[Attachments](sections/attachments.md#attachments)** | - | 🚧|
| **[Documents](sections/documents.md#documents)** | - | 🚧|
| **[Uploads](sections/uploads.md#uploads)** | - | 🚧|

API libraries
-------------

If you've got an API library for Basecamp 3, just [open a Pull Request](https://github.com/basecamp/bc3-api/compare) and let us know! We'd be happy to add it here.

Conduct
-------

Please note that this project is released with a [Contributor Code of Conduct](https://github.com/basecamp/bc3-api/blob/master/CONDUCT.md). By participating in discussions about the Basecamp 3 API, you agree to abide by its terms.


License
-------

These API docs are licensed under [Creative Commons (CC BY-SA 4.0)](http://creativecommons.org/licenses/by-sa/4.0/). Please feel free to share (alike), remix, and distribute as you see fit.
