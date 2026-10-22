# Traveller

Search a catalogue of cities, mark the ones you have visited or want to visit, and open those lists on their own pages.

This is [Wan Yau](https://github.com/WanCodes)'s solution to the Smart Pension front-end take-home. The UI is a React app called **Smart traveller**. It uses the GraphQL API in this repo. The same data is also available over REST.

## What it does

| Route | Page | Behaviour |
| --- | --- | --- |
| `/` | Home | Search by city name. Each result has **Visited** and **Wish list** checkboxes. |
| `/visited` | Visited | Cities with `visited: true`. |
| `/wish-list` | Wish list | Cities with `wishlist: true`. |

Search runs when you press Enter or the search button. Matching is a case-insensitive substring of the city name. Results render as cards showing the city and its country. Toggling a checkbox sends an `updateCity` mutation and writes the result into the Apollo cache, so Visited and Wish list stay in step with Home. A progress bar shows while a request is in flight. A short alert shows if the request fails.

The top bar links the three pages and switches light and dark colour modes. In dark mode the Smart logo uses the contrast asset.

The catalogue has 500 cities (ids 0–499). Every city starts with both flags `false`, so Visited and Wish list are empty until you mark one. Submitting an empty search returns the full catalogue, because an empty name does not filter.

## Stack

Yarn 1 workspaces monorepo. Lerna 4 is configured with `useWorkspaces: true`.

**Client** — http://localhost:3000

- React 17 and TypeScript, via Create React App 4 (`react-scripts`)
- React Router 6
- Apollo Client 3, fixed to `http://localhost:4000/graphql`
- Chakra UI 1 (layout, form controls, colour mode)
- Jest and React Testing Library, through Create React App

**API** — http://localhost:4000

- Node.js, TypeScript, and Express 4
- Apollo Server 3 at `/graphql`
- REST under `/rest/cities`, with Swagger UI at `/rest`
- One in-memory list in `packages/api/src/data/cities.ts`, shared by GraphQL and REST

## Layout

```
packages/client/src
  pages/            Home, Visited, Wish list, and their tests
  components/       Top bar, city list, city card, visited/wish-list checkboxes
  graphql/          cities query and updateCity mutation
  hooks/            useCitySearch, useUpdateCity
packages/api/src
  index.ts          Express, Apollo Server, Swagger
  graphql/          schema and resolvers
  cities/           REST routes and the shared city service
  data/cities.ts    seed data
```

## Prerequisites

- Yarn 1.x. `yarn.lock` is a Yarn classic lockfile.
- Node.js 14, as declared in the root `engines` field (`node: ^14.16.0`, `npm: >=6`).

Create React App 4 uses webpack 4. On Node 17 and later, set the OpenSSL legacy provider before starting the client:

```bash
export NODE_OPTIONS=--openssl-legacy-provider
```

## Install and run

From the repository root:

```bash
yarn install
yarn start:all
```

`yarn install` installs both workspaces. With Lerna's `useWorkspaces` setting, that is the install step.

| Script | What it runs |
| --- | --- |
| `yarn start:all` | Client and API together |
| `yarn dev:all` | Client, and the API under nodemon |
| `yarn start:client` | Client only, at http://localhost:3000 |
| `yarn start:api` | Compile the API with `tsc`, then start it |
| `yarn dev:api` | API with nodemon; recompiles `src` on change |

Open:

- App — http://localhost:3000
- GraphQL — http://localhost:4000/graphql
- REST docs — http://localhost:4000/rest

The client always calls `http://localhost:4000/graphql` (`packages/client/src/index.tsx`). Start the API before searching.

## Tests

Client tests use Jest, React Testing Library, and Apollo `MockedProvider`. They cover the app heading, search, toggling a city from a result card, the Visited and Wish list pages, and snapshots.

```bash
yarn test:client       # single run (watch disabled)
yarn test:client-dev   # watch mode
```

The API has no test suite. `yarn test` inside `packages/api` is a stub that exits with an error.

## API

GraphQL and REST both call `citiesService`. A city is `{ id, name, country, visited, wishlist }`.

Name and country filters are case-insensitive substrings. `id`, `visited`, and `wishlist` match exactly. `limit` and `offset` paginate the filtered list. The client sends a name filter from Home, or a `visited` / `wishlist` filter from the other two pages, and asks for the full filtered list.

### GraphQL

```graphql
type Query {
  cities(filter: CitiesFilters, limit: Int, offset: Int): CitiesResult
  city(id: Int!): City
}

type Mutation {
  updateCity(input: CitiesMutationInput): City
}
```

`CitiesFilters` accepts `id`, `name`, `country`, `visited`, and `wishlist`. `CitiesMutationInput` accepts `id`, `visited`, and `wishlist`. `cities` returns `{ cities, total }`. An unknown id responds with 404.

Example:

```graphql
query {
  cities(filter: { name: "lon" }) {
    total
    cities { id name country visited wishlist }
  }
}

mutation {
  updateCity(input: { id: 1, visited: true, wishlist: false }) {
    id name visited wishlist
  }
}
```

### REST

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/rest/cities` | Filter and list. Query params: `id`, `name`, `country`, `visited`, `wishlist`, `limit`, `offset`. |
| `GET` | `/rest/cities/:cityId` | Fetch one city. |
| `PUT` | `/rest/cities/:cityId` | Update flags. Body: `{ "visited": true, "wishlist": false }`. |

On list requests, `visited` and `wishlist` are the strings `true` and `false`. An invalid number or boolean returns 422. A missing city returns 404.

```bash
curl "http://localhost:4000/rest/cities?name=lon&limit=5"
curl -X PUT "http://localhost:4000/rest/cities/1" \
  -H "Content-Type: application/json" \
  -d '{"visited":true,"wishlist":false}'
```

## Notes

- Updates are written onto the in-memory array and last for the life of the API process. Restarting the API restores every city to `visited: false` and `wishlist: false`.
- Checkboxes are on Home search cards. Visited and Wish list only display cities.
- `packages/client/.env` sets `TSC_COMPILE_ON_ERROR` and `ESLINT_NO_DEV_ERRORS`, so the Create React App dev server stays up when the type-checker or ESLint reports an error.
- The root `package.json` declares an MIT license.
