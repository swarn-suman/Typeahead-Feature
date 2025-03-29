# Typeahead System

- This repository contains documentation and architecture overview for a Typeahead Search System that I implemented during a two-week work trial at Mercor as a Software Engineer. 

- The actual implementation code has been omitted as it's company property, but this repo serves to demonstrate my understanding of the system architecture, design decisions, and implementation approach.

![Demo Video](/TypeaheadFeatureDemo.gif)

## Technologies Used

**Frontend**

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![TailwindCSS](https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)

**Backend**

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-092E20?style=for-the-badge&logo=django&logoColor=white)
![DRF](https://img.shields.io/badge/Django_REST_Framework-092E20?style=for-the-badge&logo=django&logoColor=white)

**Database & Cache**

![MongoDB](https://img.shields.io/badge/MongoDB-4EA94B?style=for-the-badge&logo=mongodb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

**Authentication**

![Firebase](https://img.shields.io/badge/Firebase-FFCA28?style=for-the-badge&logo=firebase&logoColor=black)
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white)

**DevOps & Infrastructure**

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![DataDog](https://img.shields.io/badge/DataDog-632CA6?style=for-the-badge&logo=datadog&logoColor=white)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=for-the-badge&logo=pytest&logoColor=white)
![pnpm](https://img.shields.io/badge/pnpm-F69220?style=for-the-badge&logo=pnpm&logoColor=white)

## Problem Statement

Recruiters on the Mercor team platform frequently perform similar searches for contractors, often repeating the same queries or making minor modifications to previous searches. Currently, there is no mechanism to leverage their search history, forcing recruiters to manually re-enter search terms each time. This leads to:

- Inefficient workflows and wasted time.
- Inconsistent search parameters across similar searches.
- Difficulty in recalling effective previous search queries.
- Reduced productivity for recruiters managing multiple roles.

### Objectives

The primary goal was to implement a "Saved Queries/Typeahead" feature that would:

1. Automatically save user search queries in the database.
2. Store both the search text and associated filters (skills, experience, etc.)
3. Personalized ranking based on frequency and recency of searches.
4. Provide typeahead suggestions from historical searches as users type in the search bar.
5. Display up to 5 relevant previous searches in a dropdown.
6. Allow keyboard navigation through suggestions (arrow keys, Tab, and Enter).
7. Handle long queries gracefully with truncation in the UI.

### Success Metrics

- **Adoption Rate:** Recruiters who used the typeahead feature.
- **Time Savings:** Reduction in time spent on search operations.
- **User Satisfaction:** Positive feedback from recruiters.
- **Performance:** Typeahead suggestions appear within 100ms of typing.
- **Technical:** Zero regression in existing search functionality.

## System Architecture

The system consisted of multiple components working together to provide a responsive and efficient typeahead search experience:

### Backend Components

- Create new MongoDB collection for saving search history.
- Implement Redis for fast typeahead functionality.
- New API endpoint for typeahead suggestions.
- Query deduplication logic to prevent near-duplicates.
- Build a service that first retrieves suggestions from Redis and falls back to MongoDB on a cache miss.

![Backend Flow](/Backend%20Flow.png)

**Query Deduplication Logic:**

Here's how I implemented the deduplication logic in the BE to prevent near-duplicates.

![Deduplication Logic](/Deduplication%20Logic.png)

#### Frontend Components

- Enhanced search bar with typeahead functionality.
- Dropdown component for displaying suggestions.
- Keyboard navigation support.
- Integration with existing search functionality.

![Frontend Flow](/Frontend%20Flow.png)

## Data Flow Explanation

1. **User Input Flow:**

- User types in the `SearchBar/TeamSearch` component.
- Input is captured and debounced (300ms) via `useDebounceValue` hook.
- `useTypeahead` hook manages:
    - Suggestion state.
    - Selection navigation.
    - Keyboard interactions.
    - API communication.
- Firebase authentication token is obtained for API requests.

2. **Suggestion Retrieval:**

- Frontend API Layer:
    - Makes authenticated GET request to `/team/typeahead?prefix={query}` .
    - Handles token refresh and request retries.
    - Manages error states.
- Backend Processing:
    - Validates Firebase token via `FirebaseTokenAuthentication` .
    - `TypeaheadAPIView` processes the request.
- Typeahead Service Layer (`services/typeahead.py`):
    - First checks Redis cache (1-hour TTL).
    - Falls back to MongoDB for cache misses.
    - Manages caching strategy. 
    - Returns up to 5 most relevant suggestions.
    - Sorts by use_count and timestamp.  

3. **User Interaction Handling:**

- Keyboard Navigation:
    - ↑/↓: Navigate through suggestions.
    - Enter: Select and execute search.
    - Tab: Complete suggestion text.
    - Escape: Close dropdown.
- Mouse Interaction:
    - Click: Select suggestion.
- Selection Processing:
    - Updates search input.
    - Applies saved `hard_filters` .
    - Triggers search execution.
    - Updates URL parameters.

4. **Search Query Management:**

- Search Execution:
    - Validates query length (minimum 3 characters).
    - Processes hard filters.
    - Executes search with parameters.
- Query Storage:
    - Automatically saves in MongoDB:
        - `user_email`
        - `query text`
        - `hard_filters`
        - `timestamp`
        - `use_count`
- Maintains rolling history (last 50 queries).

5. **Caching Strategy:**

- Redis provides fast prefix matching for typeahead.
- It caches suggestions with 1-hour TTL.
- Key format: `typeahead:{user_email}:{prefix}` . 
- Graceful degradation to MongoDB if Redis is unavailable.
- User-specific caching ensures privacy and relevance.

## Data Storage

Created a new MongoDB Collection:

1. Collection Name: `"user_search_queries"`
2. Database Name: `"typeahead"`

```
# Detailed Schema Breakdown

{
  // Required Fields
  user_email: {
    type: String,
    required: true,
    index: true              // Indexed for faster user-specific queries.
  },
  query: {
    type: String,
    required: true,
    index: true              // Indexed for faster prefix matching.
  },
  display_text: {
    type: String,
    required: true           // Truncated version of query (max 50 chars).
  },
  hard_filters: {
    type: Object,            // Stores search filters.
    default: {}
  },

  // Timestamps
  timestamp: {
    type: DateTime,
    required: true,
    index: true              // Updated everytime the query is used. Indexed for sorting by recency.
  }, 
  created_at: {
    type: DateTime,          // First creation time. Useful for analytics & tracking query history.
    required: true
  },
  updated_at: {
    type: DateTime,          // Last modification time. Changes when query filters are modified.
    required: true
  },

  // Usage Statistics - Tracks how often a query is used for sorting suggestions.
  use_count: {
    type: Integer,
    default: 1,
    required: true
  }
}
```

## API Endpoint

| Endpoint | Method | Description |
| :--- | :---: | :--- |
| `/team/search/typeahead` | GET | Get typeahead suggestions based on prefix. |

Query Parameter:

- `prefix`: string (required) - The search prefix to match against.

Sample Response:

```
{
  "suggestions": [
    {
      "query": "react developer",
      "display_text": "React Developer",
      "hardFilters": {
        "tags": ["frontend", "javascript"],
        "status": ["available"]
      },
      "timestamp": "2023-03-01T12:34:56Z"
    }
  ]
}
```

## Risks and Mitigations

![Risks & Mitigations](/Risks%20and%20Mitigations.png)


## Future Enhancements

While it was out of scope for the initial implementation, the following enhancements could be considered for future iterations:

1. Semantic similarity for better deduplication of queries.
2. **Filter-aware suggestions:** Suggest queries that are relevant to currently applied filters.
3. **Popular searches:** Show trending searches across the platform.
4. Advanced RedisSearch features like **fuzzy matching** for typo tolerance.


## Conclusion

The Typeahead feature significantly improved the efficiency of recruiters using the Mercor team platform. By leveraging search query data and implementing a high-performance caching layer with Redis, we provided a seamless, Google/Amazon-like experience where previous searches appear as suggestions without requiring any explicit action from the user.
The implementation focuses on:

1. **Performance:** Fast responses through Redis.
2. **Context preservation:** Maintaining filters with queries.
3. **Minimal backend changes:** Leveraging existing infrastructure.
4. **Graceful degradation:** Ensuring reliability through fallback mechanisms.

This approach allowed us to deliver a high-quality feature quickly while maintaining a path for future enhancements. The feature was unobtrusive yet helpful, reducing the cognitive load on recruiters and helping them quickly access their most relevant previous searches.
