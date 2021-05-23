openapi: 3.0.3
tags:
  - name: Manga
  - name: Cover
  - name: Author
  - name: Search
  - name: Auth
  - name: ScanlationGroup
  - name: Feed
  - name: CustomList
  - name: Captcha
  - name: AtHome
  - name: Legacy
  - name: Infrastructure
info:
  title: MangaDex API
  version: 5.0.11
  contact:
    name: MangaDex staff team
    email: mangadexstaff@gmail.com
  description: |-
    MangaDex is an ad-free manga reader offering high-quality images!

    This document details our API as it is right now. It is in no way a promise to never change it, although we will endeavour to publicly notify any major change.

    # Authentication

    You can login with the `/auth/login` endpoint. On success, it will return a JWT that remains valid for 15 minutes along with a session token that allows refreshing without re-authenticating for 1 month.

    # Rate limits

    The API enforces rate-limits to protect our servers against malicious and/or mistaken use. The API keeps track of the requests on an IP-by-IP basis.
    Hence, if you're on a VPN, proxy or a shared network in general, the requests of other users on this network might affect you.

    At first, a **global limit of 5 requests per second per IP address** is in effect.

    > This limit is enforced across multiple load-balancers, and thus
    is not an exact value but rather a lower-bound that we guarantee. The exact value will be somewhere in the range `[5, 5*n]` (with `n` being the number of
    load-balancers currently active). The exact value within this range will depend on the current traffic patterns we are experiencing.

    On top of this, **some endpoints are further restricted** as follows:

    | Endpoint                           | Requests per time period    | Time period in minutes |
    |------------------------------------|--------------------------   |------------------------|
    | `POST   /account/create`           | 1                           | 60                     |
    | `GET    /account/activate/{code}`  | 30                          | 60                     |
    | `POST   /account/activate/resend`  | 5                           | 60                     |
    | `POST   /account/recover`          | 5                           | 60                     |
    | `POST   /account/recover/{code}`   | 5                           | 60                     |
    | `POST   /auth/login`               | 30                          | 60                     |
    | `POST   /auth/refresh`             | 30                          | 60                     |
    | `POST   /author`                   | 10                          | 60                     |
    | `PUT    /author`                   | 10                          | 1                      |
    | `DELETE /author/{id}`              | 10                          | 10                     |
    | `POST   /captcha/solve`            | 10                          | 10                     |
    | `POST   /chapter/{id}/read`        | 300                         | 10                     |
    | `PUT    /chapter/{id}`             | 10                          | 1                      |
    | `DELETE /chapter/{id}`             | 10                          | 1                      |
    | `POST   /manga`                    | 10                          | 60                     |
    | `PUT    /manga/{id}`               | 10                          | 60                     |
    | `DELETE /manga/{id}`               | 10                          | 10                     |
    | `POST   /cover`                    | 10                          | 1                      |
    | `PUT    /cover/{id}`               | 10                          | 1                      |
    | `DELETE /cover/{id}`               | 10                          | 10                     |
    | `POST   /group`                    | 10                          | 60                     |
    | `PUT    /group/{id}`               | 10                          | 1                      |
    | `DELETE /group/{id}`               | 10                          | 10                     |
    | `GET    /at-home/server/{id}`      | 60                          | 1                      |

    Calling these endpoints will further provide details via the following headers about your remaining quotas:

    | Header                    | Description                                                                 |
    |---------------------------|-----------------------------------------------------------------------------|
    | `X-RateLimit-Limit`       | Maximal number of requests this endpoint allows per its time period         |
    | `X-RateLimit-Remaining`   | Remaining number of requests within your quota for the current time period  |
    | `X-RateLimit-Retry-After` | Timestamp of the end of the current time period, as UNIX timestamp          |

    # Captchas

    Some endpoints may require captchas to proceed, in order to slow down automated malicious traffic.
    Legitimate users might also be affected, based on the frequency of write requests or due certain endpoints being particularly sensitive to malicious use, such as user signup.

    Once an endpoint decides that a captcha needs to be solved, a 403 Forbidden response will be returned, with the error code `captcha_required_exception`.
    The sitekey needed for recaptcha to function is provided in both the `X-Captcha-Sitekey` header field, as well as in the error context, specified as `siteKey` parameter.

    The captcha result of the client can either be passed into the repeated original request with the `X-Captcha-Result` header or alternatively to the `POST /captcha/solve` endpoint.
    The time a solved captcha is remembered varies across different endpoints and can also be influenced by individual client behavior.

    Authentication is not required for the `POST /captcha/solve` endpoint, captchas are tracked both by client ip and logged in user id.
    If you are logged in, you want to send the session token along, so you validate the captcha for your client ip and user id at the same time, but it is not required.

    # Reading a chapter using the API

    ## Retrieving pages from the MangaDex@Home network

    A valid [MangaDex@Home network](https://mangadex.network) page URL is in the following format: `{server-specific base url}/{temporary access token}/{quality mode}/{chapter hash}/{filename}`

    There are currently 2 quality modes:
    - `data`: Original upload quality
    - `data-saver`: Compressed quality

    Upon fetching a chapter from the API, you will find 4 fields necessary to compute MangaDex@Home page URLs:

    | Field                        | Type     | Description                       |
    |------------------------------|----------|-----------------------------------|
    | `.data.id`                   | `string` | API Chapter ID                    |
    | `.data.attributes.hash`      | `string` | MangaDex@Home Chapter Hash        |
    | `.data.attributes.data`      | `array`  | data quality mode filenames       |
    | `.data.attributes.dataSaver` | `array`  | data-saver quality mode filenames |

    Example
    ```json
    GET /chapter/{id}

    {
      ...,
      "data": {
        "id": "e46e5118-80ce-4382-a506-f61a24865166",
        ...,
        "attributes": {
          ...,
          "hash": "e199c7d73af7a58e8a4d0263f03db660",
          "data": [
            "x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png",
            ...
          ],
          "dataSaver": [
            "x1-ab2b7c8f30c843aa3a53c29bc8c0e204fba4ab3e75985d761921eb6a52ff6159.jpg",
            ...
          ]
        }
      }
    }
    ```

    From this point you miss only the base URL to an assigned MangaDex@Home server for your client and chapter. This is retrieved via a `GET` request
    to `/at-home/server/{ chapter .data.id }`.

    Example:
    ```json
    GET /at-home/server/e46e5118-80ce-4382-a506-f61a24865166

    {
      "baseUrl": "https://abcdefg.hijklmn.mangadex.network:12345/some-token"
    }
    ```

    The full URL is the constructed as follows
    ```
    { server .baseUrl }/{ quality mode }/{ chapter .data.attributes.hash }/{ chapter .data.attributes.{ quality mode }.[*] }

    Examples

    data quality: https://abcdefg.hijklmn.mangadex.network:12345/some-token/data/e199c7d73af7a58e8a4d0263f03db660/x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png

          base url: https://abcdefg.hijklmn.mangadex.network:12345/some-token
      quality mode: data
      chapter hash: e199c7d73af7a58e8a4d0263f03db660
          filename: x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png


    data-saver quality: https://abcdefg.hijklmn.mangadex.network:12345/some-token/data-saver/e199c7d73af7a58e8a4d0263f03db660/x1-ab2b7c8f30c843aa3a53c29bc8c0e204fba4ab3e75985d761921eb6a52ff6159.jpg

          base url: https://abcdefg.hijklmn.mangadex.network:12345/some-token
      quality mode: data-saver
      chapter hash: e199c7d73af7a58e8a4d0263f03db660
          filename: x1-ab2b7c8f30c843aa3a53c29bc8c0e204fba4ab3e75985d761921eb6a52ff6159.jpg
    ```

    If the server you have been assigned fails to serve images, you are allowed to call the `/at-home/server/{ chapter id }` endpoint again to get another server.

    Whether successful or not, **please do report the result you encountered as detailed below**. This is so we can pull the faulty server out of the network.

    ## Report

    In order to keep track of the health of the servers in the network and to improve the quality of service and reliability, we ask that you call the
    MangaDex@Home report endpoint after each image you retrieve, whether successfully or not.

    It is a `POST` request against `https://api.mangadex.network/report` and expects the following payload with our example above:

    | Field                       | Type       | Description                                                                         |
    |-----------------------------|------------|-------------------------------------------------------------------------------------|
    | `url`                       | `string`   | The full URL of the image                                                           |
    | `success`                   | `boolean`  | Whether the image was successfully retrieved                                        |
    | `cached `                   | `boolean`  | `true` iff the server returned an `X-Cache` header with a value starting with `HIT` |
    | `bytes`                     | `number`   | The size in bytes of the retrieved image                                            |
    | `duration`                  | `number`   | The time in miliseconds that the complete retrieval (not TTFB) of this image took   |

    Examples herafter.

    **Success:**
    ```json
    POST https://api.mangadex.network/report
    Content-Type: application/json

    {
      "url": "https://abcdefg.hijklmn.mangadex.network:12345/some-token/data/e199c7d73af7a58e8a4d0263f03db660/x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png",
      "success": true,
      "bytes": 727040,
      "duration": 235,
      "cached": true
    }
    ```

    **Failure:**
    ```json
    POST https://api.mangadex.network/report
    Content-Type: application/json

    {
     "url": "https://abcdefg.hijklmn.mangadex.network:12345/some-token/data/e199c7d73af7a58e8a4d0263f03db660/x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png",
     "success": false,
     "bytes": 25,
     "duration": 235,
     "cached": false
    }
    ```

    While not strictly necessary, this helps us monitor the network's healthiness, and we appreciate your cooperation towards this goal. If no one reports
    successes and failures, we have no way to know that a given server is slow/broken, which eventually results in broken image retrieval for everyone.

    # Static data

    ## Manga publication demographic

    | Value            | Description               |
    |------------------|---------------------------|
    | shounen          | Manga is a Shounen        |
    | shoujo           | Manga is a Shoujo         |
    | josei            | Manga is a Josei          |
    | seinen           | Manga is a Seinen         |

    ## Manga status

    | Value            | Description               |
    |------------------|---------------------------|
    | ongoing          | Manga is still going on   |
    | completed        | Manga is completed        |
    | hiatus           | Manga is paused           |
    | cancelled        | Manga has been cancelled  |

    ## Manga reading status

    | Value            |
    |------------------|
    | reading          |
    | on_hold          |
    | plan\_to\_read   |
    | dropped          |
    | re\_reading      |
    | completed        |

    ## Manga content rating

    | Value            | Description               |
    |------------------|---------------------------|
    | safe             | Safe content              |
    | suggestive       | Suggestive content        |
    | erotica          | Erotica content           |
    | pornographic     | Pornographic content      |

    ## CustomList visibility

    | Value            | Description               |
    |------------------|---------------------------|
    | public           | CustomList is public      |
    | private          | CustomList is private     |

    ## Relationship types

    | Value            | Description                    |
    |------------------|--------------------------------|
    | manga            | Manga resource                 |
    | chapter          | Chapter resource               |
    | cover_art        | A Cover Art for a manga `*`    |
    | author           | Author resource                |
    | artist           | Author resource (drawers only) |
    | scanlation_group | ScanlationGroup resource       |
    | tag              | Tag resource                   |
    | user             | User resource                  |
    | custom_list      | CustomList resource            |

    `*` Note, that on manga resources you get only one cover_art resource relation marking the primary cover if there are more than one.
    By default this will be the latest volume's cover art. If you like to see all the covers for a given manga, use the cover search endpoint for your mangaId and select the one you wish to display.

    ## Manga links data

    In Manga attributes you have the `links` field that is a JSON object with some strange keys, here is how to decode this object:

    | Key   | Related site  | URL                                                                                           | URL details                                                    |
    |-------|---------------|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------|
    | al    | anilist       | https://anilist.co/manga/`{id}`                                                               | Stored as id                                                   |
    | ap    | animeplanet   | https://www.anime-planet.com/manga/`{slug}`                                                   | Stored as slug                                                 |
    | bw    | bookwalker.jp | https://bookwalker.jp/`{slug}`                                                                | Stored has "series/{id}"                                       |
    | mu    | mangaupdates  | https://www.mangaupdates.com/series.html?id=`{id}`                                            | Stored has id                                                  |
    | nu    | novelupdates  | https://www.novelupdates.com/series/`{slug}`                                                  | Stored has slug                                                |
    | kt    | kitsu.io      | https://kitsu.io/api/edge/manga/`{id}` or https://kitsu.io/api/edge/manga?filter[slug]={slug} | If integer, use id version of the URL, otherwise use slug one  |
    | amz   | amazon        | N/A                                                                                           | Stored as full URL                                             |
    | ebj   | ebookjapan    | N/A                                                                                           | Stored as full URL                                             |
    | mal   | myanimelist   | https://myanimelist.net/manga/{id}                                                            | Store as id                                                    |
    | raw   | N/A           | N/A                                                                                           | Stored as full URL, untranslated stuff URL (original language) |
    | engtl | N/A           | N/A                                                                                           | Stored as full URL, official english licenced URL              |
servers:
  - url: 'https://api.mangadex.org'
    description: MangaDex Api
paths:
  /ping:
    get:
      summary: Ping the server
      tags:
        - Infrastructure
      security: []
      responses:
        '200':
          description: Pong
          content:
            application/json:
              schema:
                type: string
                default: pong
  /manga:
    get:
      summary: Manga list
      tags:
        - Manga
        - Search
      responses:
        '200':
          description: Manga list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-search-manga
      description: Search a list of Manga.
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: string
          in: query
          name: title
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: authors
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: artists
        - schema:
            type: integer
          in: query
          name: year
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: includedTags
        - schema:
            type: string
            enum:
              - AND
              - OR
            default: AND
          in: query
          name: includedTagsMode
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: excludedTags
        - schema:
            type: string
            enum:
              - AND
              - OR
            default: OR
          in: query
          name: excludedTagsMode
        - schema:
            type: array
            items:
              type: string
              enum:
                - ongoing
                - completed
                - hiatus
                - cancelled
          in: query
          name: status
        - schema:
            type: array
            items:
              type: string
              pattern: ^[a-zA-Z\-]{2,5}$
          in: query
          name: originalLanguage
        - schema:
            type: array
            items:
              type: string
              enum:
                - shounen
                - shoujo
                - josei
                - seinen
                - none
          in: query
          name: publicationDemographic
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: Manga ids (limited to 100 per request)
        - schema:
            type: array
            items:
              type: string
              enum:
                - safe
                - suggestive
                - erotica
                - pornographic
          in: query
          name: contentRating
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: createdAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: updatedAtSince
        - schema:
            type: object
            properties:
              createdAt:
                type: string
                enum:
                  - asc
                  - desc
              updatedAt:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
      security: []
    post:
      summary: Create Manga
      operationId: post-manga
      responses:
        '200':
          description: Manga Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MangaCreate'
        description: The size of the body is limited to 16KB.
      description: Create a new Manga.
      tags:
        - Manga
  '/manga/{id}/aggregate':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Manga ID
    get:
      summary: Get Manga volumes & chapters
      security: []
      tags:
        - Manga
      parameters:
        - schema:
            type: array
            items:
              type: string
              pattern: '^[a-zA-Z\-]{2,5}$'
          in: query
          name: translatedLanguage
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    default: ok
                  volumes:
                    type: object
                    additionalProperties:
                      type: object
                      properties:
                        volume:
                          type: string
                        count:
                          type: integer
                        chapters:
                          type: object
                          additionalProperties:
                            type: object
                            properties:
                              chapter:
                                type: string
                              count:
                                type: integer
  '/manga/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Manga ID
    get:
      summary: View Manga
      tags:
        - Manga
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Manga no content
      operationId: get-manga-id
      description: View Manga.
      security: []
    put:
      summary: Update Manga
      operationId: put-manga-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MangaEdit'
        description: The size of the body is limited to 16KB.
      tags:
        - Manga
    delete:
      summary: Delete Manga
      operationId: delete-manga-id
      responses:
        '200':
          description: Manga has been deleted.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
  /auth/login:
    post:
      summary: Login
      tags:
        - Auth
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LoginResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
      operationId: post-auth-login
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Login'
        description: The size of the body is limited to 2KB.
      security: []
  /auth/check:
    get:
      summary: Check token
      tags:
        - Auth
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CheckResponse'
      operationId: get-auth-check
  /auth/logout:
    post:
      summary: Logout
      tags:
        - Auth
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LogoutResponse'
        '503':
          description: Service Unavailable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: post-auth-logout
  /auth/refresh:
    post:
      summary: Refresh token
      tags:
        - Auth
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RefreshResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RefreshResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RefreshResponse'
      operationId: post-auth-refresh
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RefreshToken'
        description: The size of the body is limited to 2KB.
      security: []
  /account/create:
    post:
      summary: Create Account
      operationId: post-account-create
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      security: []
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateAccount'
        description: The size of the body is limited to 4KB.
      tags:
        - Account
  '/account/activate/{code}':
    parameters:
      - schema:
          type: string
          pattern: '[0-9a-fA-F-]+'
        name: code
        in: path
        required: true
    get:
      summary: Activate account
      tags:
        - Account
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountActivateResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-account-activate-code
      parameters: []
      security: []
  /group:
    get:
      summary: Scanlation Group list
      tags:
        - ScanlationGroup
        - Search
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ScanlationGroupList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-search-group
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: ScanlationGroup ids (limited to 100 per request)
        - schema:
            type: string
          in: query
          name: name
      security: []
    post:
      summary: Create Scanlation Group
      operationId: post-group
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ScanlationGroupResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - ScanlationGroup
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateScanlationGroup'
        description: The size of the body is limited to 8KB.
  '/group/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Scanlation Group ID
    get:
      summary: View Scanlation Group
      tags:
        - ScanlationGroup
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ScanlationGroupResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: ScanlationGroup not found
      operationId: get-group-id
      security: []
    put:
      summary: Update Scanlation Group
      operationId: put-group-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ScanlationGroupResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - ScanlationGroup
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ScanlationGroupEdit'
        description: The size of the body is limited to 8KB.
    delete:
      summary: Delete Scanlation Group
      operationId: delete-group-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - ScanlationGroup
  '/group/{id}/follow':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    post:
      summary: Follow Scanlation Group
      operationId: post-group-id-follow
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - ScanlationGroup
    delete:
      summary: Unfollow Scanlation Group
      operationId: delete-group-id-follow
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - ScanlationGroup
  /list:
    post:
      summary: Create CustomList
      operationId: post-list
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomListResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - CustomList
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CustomListCreate'
        description: The size of the body is limited to 8KB.
  '/list/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: CustomList ID
    get:
      summary: Get CustomList
      tags:
        - CustomList
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomListResponse'
        '404':
          description: CustomList not found
      operationId: get-list-id
    put:
      summary: Update CustomList
      operationId: put-list-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomListResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - CustomList
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CustomListEdit'
      description: The size of the body is limited to 8KB.
    delete:
      summary: Delete CustomList
      operationId: delete-list-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - CustomList
  '/manga/{id}/list/{listId}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Manga ID
      - schema:
          type: string
          format: uuid
        name: listId
        in: path
        required: true
        description: CustomList ID
    post:
      summary: Add Manga in CustomList
      operationId: post-manga-id-list-listId
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
        - CustomList
    delete:
      summary: Remove Manga in CustomList
      operationId: delete-manga-id-list-listId
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - CustomList
        - Manga
  /user/list:
    get:
      summary: Get logged User CustomList list
      tags:
        - CustomList
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomListList'
      operationId: get-user-list
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
      description: This will list public and private CustomList
  '/user/{id}/list':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: User ID
    get:
      summary: Get User's CustomList list
      tags:
        - CustomList
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomListList'
      operationId: get-user-id-list
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
      description: This will list only public CustomList
  '/user/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: User ID
    get:
      summary: Get User
      tags:
        - User
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
      operationId: get-user-id
  /chapter:
    parameters: []
    get:
      summary: Chapter list
      description: Chapter list. If you want the Chapters of a given Manga, please check the feed endpoints.
      operationId: get-chapter
      responses:
        '200':
          description: Chapter list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: Chapter ids (limited to 100 per request)
        - schema:
            type: string
          in: query
          name: title
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: groups
        - schema:
            type: string
            format: uuid
          in: query
          name: uploader
        - schema:
            type: string
            format: uuid
          in: query
          name: manga
        - schema:
            type: string
          in: query
          name: volume
        - schema:
            type: string
          in: query
          name: chapter
        - schema:
            type: array
            items:
              type: string
              pattern: '^[a-zA-Z\-]{2,5}$'
          in: query
          name: translatedLanguage
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: createdAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: updatedAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: publishAtSince
        - schema:
            type: object
            properties:
              createdAt:
                type: string
                enum:
                  - asc
                  - desc
              updatedAt:
                type: string
                enum:
                  - asc
                  - desc
              publishAt:
                type: string
                enum:
                  - asc
                  - desc
              volume:
                type: string
                enum:
                  - asc
                  - desc
              chapter:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
      tags:
        - Chapter
        - Search
      security: []
  '/chapter/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Chapter ID
    get:
      summary: Get Chapter
      tags:
        - Chapter
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterResponse'
              examples:
                example-1:
                  value:
                    result: ok
                    data:
                      id: 497f6eca-6276-4993-bfeb-53cbbbba6f08
                      type: chapter
                      attributes:
                        title: string
                        volume: null
                        chapter: null
                        translatedLanguage: jp
                        data:
                          - x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png
                          - x2-fc7c198880083b053bf4e8aebfc0eec1adbe52878a6c5ff08d25544a1d5502ef.png
                          - x3-90f15bc8b91deb0dc88473b532e42a99f93ee9e2c8073795c81b01fff428af80.png
                        dataSaver:
                          - x1-ab2b7c8f30c843aa3a53c29bc8c0e204fba4ab3e75985d761921eb6a52ff6159.jpg
                          - x2-3e057d937e01696adce2ac2865f62f6f6a15f03cef43d929b88e99a4b8482e03.jpg
                          - x3-128742088f99806b022bbc8006554456f2a20d0d176d7f3264a65ff9a549d0dd.jpg
                        uploader: 4df808f4-cdf8-4d1c-80e6-4af8e6ce09b8
                        version: 1
                        groups:
                          - 497f6eca-6276-4993-bfeb-53cbbbba6f08
                        manga: e7116dd3-e4ad-4d10-a3ef-6a64d730c341
                    relationships:
                      - id: 497f6eca-6276-4993-bfeb-53cbbbba6f08
                        type: string
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Chapter not found
      operationId: get-chapter-id
      security: []
    put:
      summary: Update Chapter
      operationId: put-chapter-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ChapterEdit'
        description: The size of the body is limited to 32KB.
      tags:
        - Chapter
    delete:
      summary: Delete Chapter
      operationId: delete-chapter-id
      responses:
        '200':
          description: Chapter has been deleted.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Chapter
  /user/follows/manga/feed:
    get:
      summary: Get logged User followed Manga feed
      tags:
        - Manga
        - Feed
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: User not Found
      operationId: get-user-follows-manga-feed
      parameters:
        - schema:
            type: integer
            default: 100
            minimum: 1
            maximum: 500
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              pattern: '^[a-zA-Z\-]{2,5}$'
          in: query
          name: translatedLanguage
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: createdAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: updatedAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: publishAtSince
        - schema:
            type: object
            properties:
              volume:
                type: string
                enum:
                  - asc
                  - desc
              chapter:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
  '/list/{id}/feed':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    get:
      summary: CustomList Manga feed
      tags:
        - CustomList
        - Feed
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-list-id-feed
      parameters:
        - schema:
            type: integer
            default: 100
            minimum: 1
            maximum: 500
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              pattern: '^[a-zA-Z\-]{2,5}$'
          in: query
          name: translatedLanguage
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: createdAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: updatedAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: publishAtSince
        - schema:
            type: object
            properties:
              volume:
                type: string
                enum:
                  - asc
                  - desc
              chapter:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
  '/manga/{id}/follow':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    delete:
      summary: Unfollow Manga
      operationId: delete-manga-id-follow
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
    post:
      summary: Follow Manga
      operationId: post-manga-id-follow
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
  /cover:
    get:
      summary: CoverArt list
      tags:
        - Cover
        - Search
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CoverList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-cover
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: manga
          description: Manga ids (limited to 100 per request)
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: Covers ids (limited to 100 per request)
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: uploaders
          description: User ids (limited to 100 per request)
        - schema:
            type: object
            properties:
              createdAt:
                type: string
                enum:
                  - asc
                  - desc
              updatedAt:
                type: string
                enum:
                  - asc
                  - desc
              volume:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
      security: []
  /cover/{mangaId}:
    post:
      summary: Upload Cover
      tags:
        - Cover
        - Upload
      operationId: upload-cover
      parameters:
        - schema:
            type: string
            format: uuid
          name: mangaId
          in: path
          required: true
      requestBody:
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CoverResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  /cover/{coverId}:
    parameters:
      - schema:
          type: string
          format: uuid
        name: coverId
        in: path
        required: true
    put:
      summary: Edit Cover
      tags:
        - Cover
      operationId: edit-cover
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CoverEdit'
        description: The size of the body is limited to 2KB.
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CoverResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      security: []
    delete:
      summary: Delete Cover
      tags:
        - Cover
      operationId: delete-cover
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      security: []
  /author:
    get:
      summary: Author list
      tags:
        - Author
        - Search
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthorList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-author
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: Author ids (limited to 100 per request)
        - schema:
            type: string
          in: query
          name: name
        - schema:
            type: object
            properties:
              name:
                type: string
                enum: ['asc', 'desc']
          in: query
          name: order
      security: []
    post:
      summary: Create Author
      operationId: post-author
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthorResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Author
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AuthorCreate'
        description: The size of the body is limited to 2KB.
  '/author/{id}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Author ID
    get:
      summary: Get Author
      tags:
        - Author
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Author not found
      operationId: get-author-id
      security: []
    put:
      summary: Update Author
      operationId: put-author-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthorResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AuthorEdit'
        description: The size of the body is limited to 2KB.
      tags:
        - Author
    delete:
      summary: Delete Author
      operationId: delete-author-id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Author
  /legacy/mapping:
    post:
      summary: Legacy ID mapping
      operationId: post-legacy-mapping
      responses:
        '200':
          description: This response will give you an array of mappings of resource identifiers ; the `data.attributes.newId` field corresponds to the new UUID.
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/MappingIdResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Legacy
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MappingIdBody'
        description: The size of the body is limited to 10KB.
      security: []
  '/manga/{id}/feed':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
        description: Manga ID
    get:
      summary: Manga feed
      tags:
        - Manga
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChapterList'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-manga-id-feed
      parameters:
        - schema:
            type: integer
            default: 100
            minimum: 1
            maximum: 500
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
        - schema:
            type: array
            items:
              type: string
              pattern: '^[a-zA-Z\-]{2,5}$'
          in: query
          name: translatedLanguage
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: createdAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: updatedAtSince
        - schema:
            type: string
            description: "DateTime string with following format: YYYY-MM-DDTHH:MM:SS"
            pattern: ^\d{4}-[0-1]\d-([0-2]\d|3[0-1])T([0-1]\d|2[0-3]):[0-5]\d:[0-5]\d$
          in: query
          name: publishAtSince
        - schema:
            type: object
            properties:
              volume:
                type: string
                enum:
                  - asc
                  - desc
              chapter:
                type: string
                enum:
                  - asc
                  - desc
          in: query
          name: order
      security: []
  '/manga/{id}/read':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    get:
      summary: Manga read markers
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    enum:
                      - ok
                  data:
                    type: array
                    items:
                      type: string
                      format: uuid
              examples:
                example-1:
                  value:
                    result: ok
                    data:
                      - 00057883-357b-4734-9469-52967e59ef4c
                      - 000b7978-d9bd-49ec-a8f6-a0737368415f
                      - 0015b621-a175-47f5-81fb-5976c88e18c4
      operationId: get-manga-chapter-readmarkers
      description: A list of chapter ids that are marked as read for the specified manga
      tags:
        - Manga
  '/manga/read':
    get:
      summary: Manga read markers
      parameters:
        - schema:
            type: array
            items:
              type: string
              format: uuid
          in: query
          name: ids
          description: Manga ids
          required: true
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    enum:
                      - ok
                  data:
                    type: array
                    items:
                      type: string
                      format: uuid
              examples:
                example-1:
                  value:
                    result: ok
                    data:
                      - 00057883-357b-4734-9469-52967e59ef4c
                      - 000b7978-d9bd-49ec-a8f6-a0737368415f
                      - 0015b621-a175-47f5-81fb-5976c88e18c4
      operationId: get-manga-chapter-readmarkers-2
      description: A list of chapter ids that are marked as read for the given manga ids
      tags:
        - Manga
  '/chapter/{id}/read':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    post:
      summary: Mark Chapter read
      tags:
        - Chapter
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    enum:
                      - ok
                      - error
      operationId: chapter-id-read
      description: Mark chapter as read for the current user
    delete:
      summary: Mark Chapter unread
      tags:
        - Chapter
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    enum:
                      - ok
                      - error
      operationId: chapter-id-unread
      description: Mark chapter as unread for the current user
  /manga/random:
    get:
      summary: Get a random Manga
      tags:
        - Manga
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaResponse'
      operationId: get-manga-random
      parameters: []
      security: []
  '/at-home/server/{chapterId}':
    parameters:
      - schema:
          type: string
          format: uuid
        name: chapterId
        in: path
        required: true
        description: Chapter ID
    get:
      summary: Get MangaDex@Home server URL
      tags:
        - AtHome
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  baseUrl:
                    type: string
                    description: |-
                      The base URL to construct final image URLs from.
                      The URL returned is valid for the requested chapter only, and for a duration of 15 minutes from the time of the response.
                example:
                  baseUrl: https://abcdef.xyz123.mangadex.network:12345/some-temporary-access-token
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: get-at-home-server-chapterId
      parameters:
        - schema:
            type: boolean
            default: false
          in: query
          name: forcePort443
          description: |-
            Force selecting from MangaDex@Home servers that use the standard HTTPS port 443.

            While the conventional port for HTTPS traffic is 443 and servers are encouraged to use it, it is not a hard requirement as it technically isn't
            anything special.

            However, some misbehaving school/office network will at time block traffic to non-standard ports, and setting this flag to `true` will ensure
            selection of a server that uses these.
      security: []
  /manga/tag:
    get:
      summary: Tag list
      tags:
        - Manga
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/TagResponse'
      operationId: get-manga-tag
      security: []
  /account/activate/resend:
    post:
      summary: Resend Activation code
      operationId: post-account-activate-resend
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountActivateResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Account
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SendAccountActivationCode'
        description: The size of the body is limited to 1KB.
      security: []
  /account/recover:
    post:
      summary: Recover given Account
      operationId: post-account-recover
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountActivateResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: ''
      tags:
        - Account
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SendAccountActivationCode'
        description: The size of the body is limited to 1KB.
      security: []
  '/account/recover/{code}':
    parameters:
      - schema:
          type: string
        name: code
        in: path
        required: true
    post:
      summary: Complete Account recover
      operationId: post-account-recover-code
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountActivateResponse'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Account
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RecoverCompleteBody'
        description: The size of the body is limited to 2KB.
      security: []
  /user/me:
    get:
      summary: Logged User details
      tags:
        - User
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
      operationId: get-user-me
  /user/follows/group:
    get:
      summary: Get logged User followed Groups
      tags:
        - ScanlationGroup
        - User
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ScanlationGroupList'
      operationId: get-user-follows-group
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
  /user/follows/user:
    get:
      summary: Get logged User followed User list
      tags:
        - User
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
      operationId: get-user-follows-user
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
  /user/follows/manga:
    get:
      summary: Get logged User followed Manga list
      tags:
        - Manga
        - User
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MangaList'
      operationId: get-user-follows-manga
      parameters:
        - schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
          in: query
          name: limit
        - schema:
            type: integer
            minimum: 0
          in: query
          name: offset
  /manga/status:
    get:
      summary: Get all Manga reading status for logged User
      operationId: get-manga-status
      tags:
        - Manga
      parameters:
        - schema:
            type: string
            enum:
              - reading
              - on_hold
              - plan_to_read
              - dropped
              - re_reading
              - completed
          in: query
          name: status
          description: Used to filter the list by given status
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    default: ok
                  statuses:
                    type: object
                    additionalProperties:
                      type: string
                      enum:
                        - reading
                        - on_hold
                        - plan_to_read
                        - dropped
                        - re_reading
                        - completed
              examples:
                unfiltered:
                  summary: /manga/status
                  value:
                    result: ok
                    statuses:
                      b019ea5d-5fe6-44d4-abbc-f546f210884d: reading
                      2394a5c7-1d2e-461f-acde-18726b9e37d6: dropped
                filtered:
                  summary: /manga/status?status=reading
                  value:
                    result: ok
                    statuses:
                      b019ea5d-5fe6-44d4-abbc-f546f210884d: reading
  '/manga/{id}/status':
    parameters:
      - schema:
          type: string
          format: uuid
        name: id
        in: path
        required: true
    get:
      summary: Get a Manga reading status
      operationId: get-manga-id-status
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    default: ok
                  status:
                    type: string
                    enum:
                      - reading
                      - on_hold
                      - plan_to_read
                      - dropped
                      - re_reading
                      - completed
                example:
                  result: ok
                  status: reading
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
    post:
      summary: Update Manga reading status
      operationId: post-manga-id-status
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Response'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Not Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      tags:
        - Manga
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateMangaStatus'
        description: Using a `null` value in `status` field will remove the Manga reading status. The size of the body is limited to 2KB.
  /captcha/solve:
    post:
      summary: Solve Captcha
      tags:
        - Captcha
      responses:
        '200':
          description: 'OK: Captcha has been solved'
          content:
            application/json:
              schema:
                type: object
                properties:
                  result:
                    type: string
                    enum:
                      - ok
                      - error
        '400':
          description: 'Bad Request: Captcha challenge result was wrong, the Captcha Verification service was down or other, refer to the error message and the errorCode inside the error context'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      operationId: post-captcha-solve
      requestBody:
        content:
          application/json:
            schema:
              description: ''
              type: object
              properties:
                captchaChallenge:
                  type: string
                  minLength: 1
              required:
                - captchaChallenge
            examples:
              example-1:
                value:
                  captchaChallenge: string
      description: |-
        Captchas can be solved explicitly through this endpoint, another way is to add a `X-Captcha-Result` header to any request. The same logic will verify the captcha and is probably more convenient because it takes one less request.

        Authentication is optional. Captchas are tracked for both the client ip and for the user id, if you are logged in you want to send your session token but that is not required.
      security:
        - Bearer: []
components:
  schemas:
    MangaRequest:
      description: ''
      type: object
      title: MangaRequest
      properties:
        title:
          $ref: '#/components/schemas/LocalizedString'
        altTitles:
          type: array
          items:
            $ref: '#/components/schemas/LocalizedString'
        description:
          $ref: '#/components/schemas/LocalizedString'
        authors:
          type: array
          items:
            type: string
            format: uuid
        artists:
          type: array
          items:
            type: string
            format: uuid
        links:
          type: object
          additionalProperties:
            type: string
        originalLanguage:
          type: string
          pattern: '^[a-zA-Z\-]{2,5}$'
        lastVolume:
          type: string
          nullable: true
        lastChapter:
          type: string
          nullable: true
        publicationDemographic:
          type: string
          nullable: true
          enum:
            - shounen
            - shoujo
            - josei
            - seinen
        status:
          type: string
          nullable: true
          enum:
            - ongoing
            - completed
            - hiatus
            - cancelled
        year:
          type: integer
          nullable: true
          maximum: 9999
          minimum: 1
        contentRating:
          type: string
          nullable: true
          enum:
            - safe
            - suggestive
            - erotica
            - pornographic
        modNotes:
          type: string
          nullable: true
        version:
          type: integer
          minimum: 1
    LocalizedString:
      type: object
      title: LocalizedString
      additionalProperties:
        type: string
        pattern: "^[a-z]{2,8}$"
    MangaResponse:
      title: MangaResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
        data:
          $ref: '#/components/schemas/Manga'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    ChapterResponse:
      title: ChapterResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
        data:
          $ref: '#/components/schemas/Chapter'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    Relationship:
      title: Relationship
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
    Chapter:
      title: Chapter
      type: object
      x-examples:
        example:
          id: a05de006-df86-4fe1-9d20-32664a78c1cc
          type: chapter
          attributes:
            title: Chapter title
            volume: 1
            chapter: '15.1'
            uploader:
              id: 23d52040-3a77-4ffe-84dd-6dbf259e32ed
              type: user
              attributes: []
            manga:
              id: 439ea5a9-e922-44d1-b9c1-a3e1158bd55f
              type: manga
              attributes: []
            groups:
              - id: cb4cf9ac-ee21-486f-ab0f-21b17ba0ab7f
                type: scanlation_group
                attributes: []
            translatedLanguage: en
            data:
              - x1-b765e86d5ecbc932cf3f517a8604f6ac6d8a7f379b0277a117dc7c09c53d041e.png
              - x2-fc7c198880083b053bf4e8aebfc0eec1adbe52878a6c5ff08d25544a1d5502ef.png
              - x3-90f15bc8b91deb0dc88473b532e42a99f93ee9e2c8073795c81b01fff428af80.png
            dataSaver:
              - x1-ab2b7c8f30c843aa3a53c29bc8c0e204fba4ab3e75985d761921eb6a52ff6159.jpg
              - x2-3e057d937e01696adce2ac2865f62f6f6a15f03cef43d929b88e99a4b8482e03.jpg
              - x3-128742088f99806b022bbc8006554456f2a20d0d176d7f3264a65ff9a549d0dd.jpg
            version: 1
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - chapter
        attributes:
          $ref: '#/components/schemas/ChapterAttributes'
    Manga:
      title: Manga
      type: object
      x-examples:
        example:
          id: a05de006-df86-4fe1-9d20-32664a78c1cc
          type: manga
          attributes:
            title:
              en: English title
              de: Deutscher Titel
              jp: 日本語題名
            altTitles:
              - en: Secondary english title
              - en: Third english title
              - romaji: Romaji title
            description:
              en: Nice description
            isLocked: false
            originalLanguage: jp
            year: 2020
            version: 1
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - manga
        attributes:
          $ref: '#/components/schemas/MangaAttributes'
    ErrorResponse:
      title: ErrorResponse
      type: object
      properties:
        result:
          type: string
          default: error
        errors:
          type: array
          items:
            $ref: '#/components/schemas/Error'
    Error:
      title: Error
      type: object
      properties:
        id:
          type: string
        status:
          type: integer
        title:
          type: string
        detail:
          type: string
    ChapterAttributes:
      title: ChapterAttributes
      type: object
      properties:
        title:
          type: string
          maxLength: 255
        volume:
          type: string
          nullable: true
        chapter:
          type: string
          nullable: true
          maxLength: 8
        translatedLanguage:
          type: string
          pattern: '^[a-zA-Z\-]{2,5}$'
        hash:
          type: string
        data:
          type: array
          items:
            type: string
        dataSaver:
          type: array
          items:
            type: string
        uploader:
          type: string
          format: uuid
        version:
          type: integer
          minimum: 1
        createdAt:
          type: string
        updatedAt:
          type: string
        publishAt:
          type: string
    MangaAttributes:
      title: MangaAttributes
      type: object
      properties:
        title:
          $ref: '#/components/schemas/LocalizedString'
        altTitles:
          type: array
          items:
            $ref: '#/components/schemas/LocalizedString'
        description:
          $ref: '#/components/schemas/LocalizedString'
        isLocked:
          type: boolean
        links:
          type: object
          additionalProperties:
            type: string
        originalLanguage:
          type: string
        lastVolume:
          type: string
          nullable: true
        lastChapter:
          type: string
          nullable: true
        publicationDemographic:
          type: string
          nullable: true
        status:
          type: string
          nullable: true
        year:
          type: integer
          nullable: true
        contentRating:
          type: string
          nullable: true
        tags:
          type: array
          items:
            $ref: '#/components/schemas/Tag'
        version:
          type: integer
          minimum: 1
        createdAt:
          type: string
        updatedAt:
          type: string
    MangaCreate:
      allOf:
        - $ref: '#/components/schemas/MangaRequest'
        - required:
            - title
    MangaEdit:
      allOf:
        - $ref: '#/components/schemas/MangaRequest'
        - required:
            - version
    ChapterEdit:
      allOf:
        - $ref: '#/components/schemas/ChapterRequest'
        - required:
            - version
    Response:
      title: Response
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
    Login:
      type: object
      title: Login
      additionalProperties: false
      properties:
        username:
          type: string
          minLength: 1
          maxLength: 64
        password:
          type: string
          minLength: 8
          maxLength: 1024
      required:
        - username
        - password
    LoginResponse:
      title: LoginResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
        token:
          type: object
          properties:
            session:
              type: string
            refresh:
              type: string
    CheckResponse:
      title: CheckResponse
      type: object
      description: ''
      properties:
        ok:
          type: string
          enum:
            - ok
            - error
        isAuthenticated:
          type: boolean
        roles:
          type: array
          items:
            type: string
        permissions:
          type: array
          items:
            type: string
    LogoutResponse:
      title: LogoutResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
    RefreshToken:
      type: object
      title: RefreshToken
      additionalProperties: false
      properties:
        token:
          type: string
          minLength: 1
      required:
        - token
    RefreshResponse:
      title: RefreshResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
        token:
          type: object
          properties:
            session:
              type: string
            refresh:
              type: string
        message:
          type: string
      required:
        - result
    AccountActivateResponse:
      title: AccountActivateResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
    CreateAccount:
      title: CreateAccount
      type: object
      properties:
        username:
          type: string
          minLength: 1
          maxLength: 64
        password:
          type: string
          minLength: 8
          maxLength: 1024
        email:
          type: string
          format: email
      required:
        - username
        - password
        - email
    ScanlationGroupResponse:
      title: ScanlationGroupResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
        data:
          $ref: '#/components/schemas/ScanlationGroup'
        relationships:
          type: array
          items:
            type: object
            properties:
              id:
                type: string
                format: uuid
              type:
                type: string
    ScanlationGroup:
      title: ScanlationGroup
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - scanlation_group
        attributes:
          $ref: '#/components/schemas/ScanlationGroupAttributes'
    ScanlationGroupAttributes:
      title: ScanlationGroupAttributes
      type: object
      properties:
        name:
          type: string
        leader:
          $ref: '#/components/schemas/User'
        version:
          type: integer
          minimum: 1
        createdAt:
          type: string
        updatedAt:
          type: string
    User:
      title: User
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - user
        attributes:
          $ref: '#/components/schemas/UserAttributes'
    UserAttributes:
      title: UserAttributes
      type: object
      properties:
        username:
          type: string
        version:
          type: integer
          minimum: 1
    CreateScanlationGroup:
      title: CreateScanlationGroup
      type: object
      properties:
        name:
          type: string
        leader:
          type: string
          format: uuid
        members:
          type: array
          items:
            type: string
            format: uuid
        version:
          type: integer
          minimum: 1
      required:
        - name
    ScanlationGroupEdit:
      title: ScanlationGroupEdit
      type: object
      properties:
        name:
          type: string
        leader:
          type: string
          format: uuid
        members:
          type: array
          items:
            type: string
            format: uuid
        version:
          type: integer
          minimum: 1
      required:
        - version
    CustomListCreate:
      title: CustomListCreate
      type: object
      properties:
        name:
          type: string
        visibility:
          type: string
          enum:
            - public
            - private
        manga:
          type: array
          items:
            type: string
            format: uuid
        version:
          type: integer
          minimum: 1
      required:
        - name
    CustomListEdit:
      title: CustomListEdit
      type: object
      properties:
        name:
          type: string
        visibility:
          type: string
          enum:
            - public
            - private
        manga:
          type: array
          items:
            type: string
            format: uuid
        version:
          type: integer
          minimum: 1
      required:
        - version
    CustomListResponse:
      title: CustomListResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
            - error
        data:
          $ref: '#/components/schemas/CustomList'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    CustomList:
      title: CustomList
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - custom_list
        attributes:
          $ref: '#/components/schemas/CustomListAttributes'
    CustomListAttributes:
      title: CustomListAttributes
      type: object
      properties:
        name:
          type: string
        visibility:
          type: string
          enum:
            - private
            - public
        owner:
          $ref: '#/components/schemas/User'
        version:
          type: integer
          minimum: 1
    CoverResponse:
      title: CoverResponse
      type: object
      properties:
        result:
          type: string
        data:
          $ref: '#/components/schemas/Cover'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    Cover:
      title: Cover
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - cover_art
        attributes:
          $ref: '#/components/schemas/CoverAttributes'
    CoverAttributes:
      title: CoverAttributes
      type: object
      properties:
        volume:
          type: string
          nullable: true
        fileName:
          type: string
        description:
          type: string
          nullable: true
        version:
          type: integer
          minimum: 1
        createdAt:
          type: string
        updatedAt:
          type: string
    CoverEdit:
      title: CoverEdit
      type: object
      properties:
        volume:
          type: string
          nullable: true
          minLength: 0
          maxLength: 8
        description:
          type: string
          nullable: true
          minLength: 0
          maxLength: 512
        version:
          type: integer
          minimum: 1
      required:
        - version
        - volume
    AuthorResponse:
      title: AuthorResponse
      type: object
      properties:
        result:
          type: string
        data:
          $ref: '#/components/schemas/Author'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    Author:
      title: Author
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - author
        attributes:
          $ref: '#/components/schemas/AuthorAttributes'
    AuthorAttributes:
      title: AuthorAttributes
      type: object
      properties:
        name:
          type: string
        imageUrl:
          type: string
        biography:
          type: object
          additionalProperties:
            type: string
        version:
          type: integer
          minimum: 1
        createdAt:
          type: string
        updatedAt:
          type: string
    AuthorEdit:
      title: AuthorEdit
      type: object
      properties:
        name:
          type: string
        version:
          type: integer
          minimum: 1
      required:
        - version
    AuthorCreate:
      type: object
      title: AuthorCreate
      additionalProperties: false
      properties:
        name:
          type: string
        version:
          type: integer
          minimum: 1
      required:
        - name
    MappingIdBody:
      type: object
      title: MappingIdBody
      additionalProperties: false
      properties:
        type:
          type: string
          enum:
            - group
            - manga
            - chapter
            - tag
        ids:
          type: array
          items:
            type: integer
    MappingIdResponse:
      title: MappingIdResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
        data:
          $ref: '#/components/schemas/MappingId'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    MappingId:
      title: MappingId
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - mapping_id
        attributes:
          $ref: '#/components/schemas/MappingIdAttributes'
    MappingIdAttributes:
      title: MappingIdAttributes
      type: object
      properties:
        type:
          type: string
          enum:
            - manga
            - chapter
            - group
            - tag
        legacyId:
          type: integer
        newId:
          type: string
          format: uuid
    TagResponse:
      title: TagResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
        data:
          $ref: '#/components/schemas/Tag'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    Tag:
      title: Tag
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum:
            - tag
        attributes:
          $ref: '#/components/schemas/TagAttributes'
    TagAttributes:
      title: TagAttributes
      type: object
      properties:
        name:
          $ref: '#/components/schemas/LocalizedString'
        description:
          $ref: '#/components/schemas/LocalizedString'
        group:
          type: string
        version:
          type: integer
          minimum: 1
    UserResponse:
      title: UserResponse
      type: object
      properties:
        result:
          type: string
          enum:
            - ok
        data:
          $ref: '#/components/schemas/User'
        relationships:
          type: array
          items:
            $ref: '#/components/schemas/Relationship'
    SendAccountActivationCode:
      type: object
      title: SendAccountActivationCode
      additionalProperties: false
      properties:
        email:
          type: string
          format: email
      required:
        - email
    RecoverCompleteBody:
      type: object
      title: RecoverCompleteBody
      additionalProperties: false
      properties:
        newPassword:
          type: string
          minLength: 8
          maxLength: 1024
      required:
        - newPassword
    UpdateMangaStatus:
      title: UpdateMangaStatus
      type: object
      properties:
        status:
          type: string
          nullable: true
          enum:
            - reading
            - on_hold
            - plan_to_read
            - dropped
            - re_reading
            - completed
      required:
        - status
    ChapterRequest:
      title: ChapterRequest
      type: object
      properties:
        title:
          type: string
          maxLength: 255
        volume:
          type: string
          nullable: true
        chapter:
          type: string
          nullable: true
          maxLength: 8
        translatedLanguage:
          type: string
          pattern: '^[a-zA-Z\-]{2,5}$'
        data:
          type: array
          items:
            type: string
        dataSaver:
          type: array
          items:
            type: string
        version:
          type: integer
          minimum: 1
    CoverList:
      title: CoverList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/CoverResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    AuthorList:
      title: AuthorList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/AuthorResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    ChapterList:
      title: ChapterList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/ChapterResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    ScanlationGroupList:
      title: ScanlationGroupList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/ScanlationGroupResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    MangaList:
      title: MangaList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/MangaResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    CustomListList:
      title: CustomListList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/CustomListResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
    UserList:
      title: UserList
      type: object
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/UserResponse'
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer
  securitySchemes:
    Bearer:
      type: http
      scheme: bearer
security:
  - Bearer: []
