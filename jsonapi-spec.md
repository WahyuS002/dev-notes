---
title: JSON:API
topic: API Specification
tags: [jsonapi, api, spec]
---

# JSON:API Notes

## 1. What is JSON:API?

[JSON:API](https://jsonapi.org/) is a specification based on [JavaScript Object Notation (JSON)](https://datatracker.ietf.org/doc/html/rfc8259).  
Unlike plain JSON (with media type `application/json`), JSON:API defines its own media type: `application/vnd.api+json`. This media type is used by both clients (when making requests) and servers (when sending responses).

---

### 1.1. Media Types in HTTP

In HTTP, it is standard practice for the server to include a `Content-Type` header in its response.
This header tells the client how to interpret the returned content.
The general syntax is:

```http
Content-Type: <media-type>
```

Here's an example of a JSON:API response:

**HTTP Headers:**

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json
```

**Response Body:**

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "TypeScript Basics",
      "author": "Jane Smith",
      "published": "2024-01-15"
    }
  }
}
```

## 2. JSON:API Top Level

### 2.1. Required Members (at least one must be present)

Every JSON:API response must have at least one of these main sections:

- **`data`**: Contains the actual content you requested (like user info, articles, etc.)
- **`errors`**: Contains error details when something goes wrong
- **`meta`**: Additional information that doesn't fit elsewhere (like pagination info, request metadata)

**Important rule:** `data` and `errors` can't exist together - a response either succeeds (has `data`) or fails (has `errors`), never both.

### 2.2. Optional Members

A response can also include these optional sections:

- **`jsonapi`**: Information about the server's JSON:API version/features
- **`links`**: URLs related to the response (see details below)
- **`included`**: Related resources that were requested along with the main data (to reduce API calls)

**Note:** `included` can only be present when `data` exists - it doesn't make sense to have related resources without main data.

#### 2.3. The `links` Object

The top-level `links` object contains URLs related to the entire response. Common links include:

- **`self`**: The URL that generated this exact response (including all query parameters like filters, sorting, pagination)
- **`related`**: URL to related resources when the data represents a relationship
- **`describedby`**: Link to API documentation (OpenAPI/JSON Schema) for this endpoint
- **Pagination links**: `first`, `last`, `prev`, `next` for paginated collections

**Why `self` is useful:** It allows clients to refresh the current data without rebuilding the URL. The `self` link preserves all the original query parameters (filters, includes, sorting, etc.).

### 2.4. Examples:

**Success response with data:**

```json
{
  "data": {
    "type": "users",
    "id": "123",
    "attributes": {
      "name": "John Doe"
    }
  }
}
```

**Error response (simple):**

```json
{
  "errors": [
    {
      "status": "404",
      "title": "Not Found",
      "detail": "User with id 999 does not exist"
    }
  ]
}
```

**Error response (comprehensive):**

```json
{
  "errors": [
    {
      "id": "err-12345",
      "status": "422",
      "code": "VALIDATION_ERROR",
      "title": "Invalid Attribute",
      "detail": "The 'email' field must be a valid email address",
      "source": {
        "pointer": "/data/attributes/email",
        "parameter": null,
        "header": null
      },
      "links": {
        "about": "https://api.example.com/docs/errors/VALIDATION_ERROR",
        "type": "https://api.example.com/errors/types/validation"
      },
      "meta": {
        "timestamp": "2024-01-15T10:30:00Z",
        "request_id": "req-abc123"
      }
    },
    {
      "status": "422",
      "title": "Invalid Attribute",
      "detail": "The 'age' field must be a positive number",
      "source": {
        "pointer": "/data/attributes/age"
      }
    }
  ]
}
```

**Response with only meta:**

```json
{
  "meta": {
    "api_version": "1.0",
    "copyright": "© 2024"
  }
}
```

**Complete response with optional members:**

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "Learning JSON:API"
    },
    "relationships": {
      "author": {
        "data": { "type": "users", "id": "42" }
      }
    }
  },
  "included": [
    {
      "type": "users",
      "id": "42",
      "attributes": {
        "name": "John Doe"
      }
    }
  ],
  "links": {
    "self": "https://api.example.com/articles/1?include=author",
    "describedby": "https://api.example.com/docs#articles"
  },
  "jsonapi": {
    "version": "1.0"
  }
}
```

**Collection with pagination links:**

```json
{
  "data": [
    { "type": "articles", "id": "1", "attributes": { "title": "First" } },
    { "type": "articles", "id": "2", "attributes": { "title": "Second" } }
  ],
  "links": {
    "self": "https://api.example.com/articles?page[number]=2&page[size]=2",
    "first": "https://api.example.com/articles?page[number]=1&page[size]=2",
    "prev": "https://api.example.com/articles?page[number]=1&page[size]=2",
    "next": "https://api.example.com/articles?page[number]=3&page[size]=2",
    "last": "https://api.example.com/articles?page[number]=10&page[size]=2"
  },
  "meta": {
    "total_pages": 10,
    "current_page": 2
  }
}
```

## 3. Primary Data Structure

The `data` field (primary data) must follow specific formats depending on what you're requesting:

### 3.1. Resource Objects vs Resource Identifiers

#### 3.1.1. Resource Object

A complete representation of a resource with all its information.

**Required fields:**

- `type`: The resource type (e.g., "articles", "users")
- `id`: Unique identifier (except when creating new resources)

**Optional fields:**

- `attributes`: The resource's data (title, name, content, etc.)
- `relationships`: Links to other related resources (see details below)
- `links`: URLs related to this specific resource (see Resource Links below)
- `meta`: Any extra information about the resource

**Important naming rule:** You can't use `type` or `id` as attribute or relationship names.

##### Relationships

Relationships connect resources to each other. Each relationship is an object that must contain at least one of:

**`data` field:** The actual relationship linkage (Resource Linkage)

- **To-one relationship:** Single resource identifier or `null`
- **To-many relationship:** Array of resource identifiers (can be empty)

Resource linkage MUST be represented as:

- `null` for empty to-one relationships
- `[]` (empty array) for empty to-many relationships
- Single resource identifier object for non-empty to-one relationships
- Array of resource identifier objects for non-empty to-many relationships

**`links` field:** URLs for the relationship

- `self`: URL to manage the relationship itself (add/remove/replace)
- `related`: URL to fetch the actual related resources

**`meta` field:** Additional relationship metadata

**Example - Article with relationships:**

```json
{
  "type": "articles",
  "id": "1",
  "attributes": {
    "title": "Learning JSON:API"
  },
  "relationships": {
    "author": {
      "links": {
        "self": "/articles/1/relationships/author",
        "related": "/articles/1/author"
      },
      "data": { "type": "people", "id": "9" }
    },
    "comments": {
      "links": {
        "self": "/articles/1/relationships/comments",
        "related": "/articles/1/comments"
      },
      "data": [
        { "type": "comments", "id": "5" },
        { "type": "comments", "id": "12" }
      ],
      "meta": {
        "count": 2
      }
    },
    "tags": {
      "data": []
    }
  }
}
```

**Key points about relationships:**

- The `self` link lets you modify the relationship (e.g., change an article's author)
- The `related` link fetches the full related resources
- Empty to-many relationships use an empty array `[]`, not `null`
- To-one relationships can be `null` (no related resource)

**How Resource Linkage works with Compound Documents:**

When you include related resources (using `?include=author,comments`), the response becomes a "compound document":

1. The `data` field in relationships contains just resource identifiers (type & id)
2. The full resource objects are in the top-level `included` array
3. The client matches them by type and id - no extra API calls needed!

**Example - Compound document with included resources:**

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "Rails is Omakase"
    },
    "relationships": {
      "author": {
        "links": {
          "self": "/articles/1/relationships/author",
          "related": "/articles/1/author"
        },
        "data": { "type": "people", "id": "9" }
      }
    }
  },
  "included": [
    {
      "type": "people",
      "id": "9",
      "attributes": {
        "name": "DHH",
        "email": "dhh@example.com"
      }
    }
  ]
}
```

In this example:

- The article's author relationship points to `{ "type": "people", "id": "9" }`
- The full author details are in the `included` array
- The client can display both article and author without another request

##### Resource Links

The optional `links` member within each resource object contains URLs related to that specific resource.

Most commonly includes:

- **`self`**: The URL to fetch this specific resource directly

**Example - Resource with links:**

```json
{
  "type": "articles",
  "id": "1",
  "attributes": {
    "title": "Rails is Omakase"
  },
  "links": {
    "self": "https://example.com/articles/1"
  }
}
```

When you GET the `self` URL, the server MUST respond with this resource as the primary data. This is useful for:

- Bookmarking specific resources
- Sharing direct links to resources
- Refreshing a single resource from a collection

#### 3.1.2. Resource Identifier Object

A minimal reference to a resource - just enough to identify it.

**Required fields:**

- `type`: The resource type
- `id`: Unique identifier

**Optional fields:**

- `meta`: Additional metadata about the reference

Think of it like the difference between having someone's full contact card vs. just their name and phone number.

### 3.2. For Single Resources

Can be one of:

- **Resource object**: Full resource with all its details
- **Resource identifier**: Just the type and ID (like a reference)
- **`null`**: When the requested resource doesn't exist

### 3.3. For Collections

Must ALWAYS be an array, containing:

- **Array of resource objects**: Full resources with details
- **Array of resource identifiers**: Just references
- **Empty array `[]`**: When no resources match

**Important:** Collections are ALWAYS arrays, even if there's only one item or zero items.

### 3.4. Examples:

**Complete resource object (with all possible fields):**

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "Rails is Omakase",
      "content": "Article content here...",
      "published": "2024-01-15"
    },
    "relationships": {
      "author": {
        "links": {
          "self": "/articles/1/relationships/author",
          "related": "/articles/1/author"
        },
        "data": { "type": "people", "id": "9" }
      },
      "comments": {
        "data": [
          { "type": "comments", "id": "5" },
          { "type": "comments", "id": "12" }
        ]
      }
    },
    "links": {
      "self": "/articles/1"
    },
    "meta": {
      "views": 1523,
      "featured": true
    }
  }
}
```

**Resource identifier (minimal reference):**

```json
{
  "data": {
    "type": "articles",
    "id": "1"
  }
}
```

**Creating a new resource (no id required):**

```json
{
  "data": {
    "type": "articles",
    "attributes": {
      "title": "New Article",
      "content": "This will be created..."
    }
  }
}
```

**Collection (always an array):**

```json
{
  "data": [
    {
      "type": "articles",
      "id": "1",
      "attributes": { "title": "First Article" }
    },
    {
      "type": "articles",
      "id": "2",
      "attributes": { "title": "Second Article" }
    }
  ]
}
```

**Empty collection (still an array):**

```json
{
  "data": []
}
```

## 4. Error Objects

Error responses provide detailed information about what went wrong. Errors are always returned as an array under the `errors` key, even if there's only one error.

### 4.1. Error Object Fields

Each error object can contain these fields (at least one is required):

#### 4.1.1. Common Fields:

- **`status`**: HTTP status code as a string (e.g., "404", "422")
- **`title`**: General error message (stays the same for all occurrences of this error type)
- **`detail`**: Specific details about this particular error occurrence

#### 4.1.2. Tracking & Debugging:

- **`id`**: Unique identifier for this specific error occurrence
- **`code`**: Application-specific error code (e.g., "VALIDATION_ERROR", "AUTH_EXPIRED")

#### 4.1.3. Error Source:

The `source` object pinpoints where the error occurred:

- **`pointer`**: JSON Pointer ([RFC 6901](https://tools.ietf.org/html/rfc6901)) that shows the exact path to the problematic field in your request
- **`parameter`**: Query parameter that caused the error
- **`header`**: Request header that caused the error

**Why `pointer` is useful:**

Imagine you send this request to create a user:

```json
{
  "data": {
    "type": "users",
    "attributes": {
      "name": "John",
      "email": "invalid-email",
      "profile": {
        "age": -5,
        "phone": "123"
      }
    }
  }
}
```

Without pointers, you'd get generic errors:

```json
{
  "errors": [
    { "detail": "Invalid email format" },
    { "detail": "Age must be positive" },
    { "detail": "Phone number too short" }
  ]
}
```

With pointers, you know EXACTLY which fields have problems:

```json
{
  "errors": [
    {
      "detail": "Invalid email format",
      "source": { "pointer": "/data/attributes/email" }
    },
    {
      "detail": "Age must be positive",
      "source": { "pointer": "/data/attributes/profile/age" }
    },
    {
      "detail": "Phone number too short",
      "source": { "pointer": "/data/attributes/profile/phone" }
    }
  ]
}
```

This makes it easy for frontend apps to:

- Highlight the exact form fields with errors
- Show error messages next to the right inputs
- Map errors back to nested object fields

#### 4.1.4. Additional Information:

- **`links`**: URLs for more information
  - `about`: Detailed explanation of this specific error
  - `type`: Documentation about this error type
- **`meta`**: Any additional metadata (timestamps, request IDs, etc.)

### 4.2. Examples:

**Validation error with source:**

```json
{
  "errors": [
    {
      "status": "422",
      "title": "Validation Failed",
      "detail": "Email format is invalid",
      "source": {
        "pointer": "/data/attributes/email"
      }
    }
  ]
}
```

**Authentication error:**

```json
{
  "errors": [
    {
      "status": "401",
      "code": "TOKEN_EXPIRED",
      "title": "Authentication Failed",
      "detail": "Your session has expired. Please login again.",
      "links": {
        "about": "https://api.example.com/docs/auth#token-expiry"
      }
    }
  ]
}
```

**Multiple errors:**

```json
{
  "errors": [
    {
      "status": "422",
      "source": { "pointer": "/data/attributes/email" },
      "title": "Invalid Value",
      "detail": "Email is required"
    },
    {
      "status": "422",
      "source": { "pointer": "/data/attributes/age" },
      "title": "Invalid Value",
      "detail": "Age must be between 0 and 120"
    }
  ]
}
```

## 5. Compound Documents

Compound documents allow servers to include related resources in a single response, reducing the number of API calls needed.

### How Compound Documents Work

1. **Request with `include` parameter**: Client requests related resources using `?include=author,comments`
2. **Response structure**:
   - Primary data in `data` field (as usual)
   - Related resources in top-level `included` array
   - Relationships use resource identifiers to link to included resources

### Rules for Included Resources

- **Must be in an array**: All included resources go in the top-level `included` array
- **Must be linked**: Every included resource must be connected to the primary data through relationships (direct or indirect)
- **Full linkage required**: Can't include orphaned resources that aren't referenced anywhere
- **No duplicates**: Cannot include more than one resource object for each `type` and `id` pair
- **Exception**: Sparse fieldsets may exclude some relationship data

**Important:** Think of `type` and `id` as a composite key that uniquely identifies resources. Even if the same resource is referenced multiple times throughout the document, it should only appear once in the entire response (either in `data` or `included`). This ensures a single canonical resource object is returned with each response.

### Complete Example

Here's an article with its author and comments included:

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON:API paints my bikeshed!"
    },
    "links": {
      "self": "http://example.com/articles/1"
    },
    "relationships": {
      "author": {
        "links": {
          "self": "http://example.com/articles/1/relationships/author",
          "related": "http://example.com/articles/1/author"
        },
        "data": { "type": "people", "id": "9" }
      },
      "comments": {
        "links": {
          "self": "http://example.com/articles/1/relationships/comments",
          "related": "http://example.com/articles/1/comments"
        },
        "data": [
          { "type": "comments", "id": "5" },
          { "type": "comments", "id": "12" }
        ]
      }
    }
  },
  "included": [
    {
      "type": "people",
      "id": "9",
      "attributes": {
        "firstName": "Dan",
        "lastName": "Gebhardt",
        "twitter": "@dgeb"
      }
    },
    {
      "type": "comments",
      "id": "5",
      "attributes": {
        "body": "First comment!"
      },
      "relationships": {
        "author": {
          "data": { "type": "people", "id": "2" }
        }
      }
    },
    {
      "type": "comments",
      "id": "12",
      "attributes": {
        "body": "Another comment!"
      },
      "relationships": {
        "author": {
          "data": { "type": "people", "id": "9" }
        }
      }
    }
  ]
}
```

### Benefits

- **Single request**: Get article, author, and comments in one API call
- **Reduced latency**: No need for multiple round trips
- **Complete data**: Frontend has everything needed to render the page
- **Efficient caching**: Each resource can still be cached individually by type and id

## 6. Meta Information

The `meta` member can be used to include non-standard meta-information at various levels of a JSON:API document.

### Where `meta` can appear:

- **Top-level**: Document-wide metadata
- **Resource objects**: Metadata about specific resources
- **Resource identifier objects**: Metadata about references
- **Relationship objects**: Metadata about relationships
- **Error objects**: Metadata about errors
- **Links objects**: Metadata about links

### Rules:

- The value of each `meta` member MUST be an object (a "meta object")
- Any members MAY be specified within meta objects - it's completely flexible

### Examples:

**Top-level meta (common use cases):**

```json
{
  "data": [
    // ... resources
  ],
  "meta": {
    "copyright": "Copyright 2015 Example Corp.",
    "authors": ["Yehuda Katz", "Steve Klabnik", "Dan Gebhardt", "Tyler Kellen"],
    "total_records": 100,
    "page_size": 20,
    "current_page": 1
  }
}
```

**Resource-level meta:**

```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON:API paints my bikeshed!"
    },
    "meta": {
      "views": 12543,
      "last_updated": "2024-01-15T10:30:00Z",
      "featured": true
    }
  }
}
```

**Relationship meta:**

```json
{
  "relationships": {
    "comments": {
      "data": [
        { "type": "comments", "id": "5" },
        { "type": "comments", "id": "12" }
      ],
      "meta": {
        "total_count": 15,
        "showing": 2
      }
    }
  }
}
```

## 7. Links

Links are used throughout JSON:API to provide URLs for navigation and actions. The `links` member can appear at various levels of a document.

### Link Formats

Within a `links` object, a link MUST be represented as either:

- **String**: A simple URL string
- **Link object**: An object with additional metadata about the link
- **`null`**: When the link doesn't exist

### Link Objects

When you need more than just a URL, use a link object with these fields:

**Required:**

- `href`: The actual URL string

**Optional:**

- `rel`: The relationship type of the link
- `describedby`: Link to documentation about this link
- `title`: Human-readable label for the link (useful for UI)
- `type`: Media type of the link's target
- `hreflang`: Language(s) of the link's target
- `meta`: Additional metadata about the link

### Examples

**Simple string links:**

```json
{
  "links": {
    "self": "http://example.com/articles/1/relationships/comments",
    "related": "http://example.com/articles/1/comments"
  }
}
```

**Link objects with metadata:**

```json
{
  "links": {
    "self": "http://example.com/articles/1/relationships/comments",
    "related": {
      "href": "http://example.com/articles/1/comments",
      "title": "Comments",
      "describedby": "http://example.com/schemas/article-comments",
      "meta": {
        "count": 10
      }
    }
  }
}
```

**Internationalization with hreflang:**

```json
{
  "links": {
    "related": {
      "href": "http://example.com/articles/1/comments",
      "hreflang": ["en", "fr"],
      "type": "application/vnd.api+json"
    }
  }
}
```

### Where Links Appear

- **Top-level**: Document-wide links (pagination, self, etc.)
- **Resource objects**: Links specific to that resource
- **Relationship objects**: Links for managing and fetching relationships
- **Error objects**: Links to error documentation

### Common Link Relations

- **`self`**: Link to the current resource/document
- **`related`**: Link to related resources
- **`first`**, **`last`**, **`prev`**, **`next`**: Pagination links
- **`describedby`**: Link to documentation/schema

**Note:** The `type` and `hreflang` members are hints - the target resource isn't guaranteed to be available in the indicated media type or language when actually accessed.

## 8. Fetching Resources

This section covers how to retrieve data from a JSON:API server using GET requests.

### Required Support

A server MUST support fetching resource data for every URL provided as:

- A `self` link in top-level links
- A `self` link in resource-level links
- A `related` link in relationship-level links

### Request Format

All JSON:API requests must include the proper Accept header:

```http
GET /articles HTTP/1.1
Accept: application/vnd.api+json
```

### Common URL Patterns

**Collection of resources:**

```http
GET /articles
Accept: application/vnd.api+json
```

**Single resource:**

```http
GET /articles/1
Accept: application/vnd.api+json
```

**Related resource via relationship:**

```http
GET /articles/1/author
Accept: application/vnd.api+json
```

### Response Types

#### 200 OK - Success

**For collections** - Returns an array of resource objects (or empty array):

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json

{
  "links": {
    "self": "http://example.com/articles"
  },
  "data": [
    {
      "type": "articles",
      "id": "1",
      "attributes": {
        "title": "JSON:API paints my bikeshed!"
      }
    },
    {
      "type": "articles",
      "id": "2",
      "attributes": {
        "title": "Rails is Omakase"
      }
    }
  ]
}
```

**Empty collection:**

```json
{
  "links": {
    "self": "http://example.com/articles"
  },
  "data": []
}
```

**For single resources** - Returns a resource object or `null`:

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json

{
  "links": {
    "self": "http://example.com/articles/1"
  },
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON:API paints my bikeshed!"
    },
    "relationships": {
      "author": {
        "links": {
          "related": "http://example.com/articles/1/author"
        }
      }
    }
  }
}
```

**When a relationship is empty (to-one):**

```json
{
  "links": {
    "self": "http://example.com/articles/1/author"
  },
  "data": null
}
```

#### 404 Not Found

When a single resource doesn't exist:

```http
HTTP/1.1 404 Not Found
Content-Type: application/vnd.api+json

{
  "errors": [
    {
      "status": "404",
      "title": "Not Found",
      "detail": "Article with id '999' does not exist"
    }
  ]
}
```

**Exception:** Some APIs might return `200 OK` with `"data": null` instead of 404, especially for optional relationships.

#### Other Status Codes

Servers may respond with other HTTP status codes:

- **401 Unauthorized** - Authentication required
- **403 Forbidden** - Access denied
- **500 Internal Server Error** - Server problem

All error responses should include error objects in the `errors` array.

### Key Points

1. **Always use proper Accept header** - `application/vnd.api+json`
2. **Collections are always arrays** - Even empty ones use `[]`
3. **Missing single resources** - Usually 404, sometimes `null`
4. **Relationships can be empty** - Use `null` for to-one, `[]` for to-many
5. **Follow HTTP semantics** - Status codes have standard meanings
