---
title: "06 - PWA Studio & Hyvä"
description: "Magento 2.4.8 PWA Studio (Venia) architecture, Hyvä React-based theme, comparison of headless frontend approaches, GraphQL-only data layer, and when to use each"
tags: [magento2, pwa-studio, venia, hyva, headless, react, graphql]
rank: 6
pathways: [magento2-deep-dive]
see_also:
  - "GraphQL schema extension: _supplemental/11-graphql-rest-api.md"
  - "REST/GraphQL API: ../07-api-development/11-graphql-rest-api.md"
---

# PWA Studio & Hyvä: Headless Frontend Approaches for Magento 2.4.8

## 1. The Headless Frontend Problem

### Why Magento's Luma/Blank Theme Is Complex to Customize

Magento's default Luma theme (built on the Blank parent) presents significant customization challenges:

- **KnockoutJS templates** — Require custom binding handlers and are difficult to debug
- **XML layout overrides** — Cascading XML files make changes hard to trace
- ** Knockout-based checkout** — One of the most complex areas to customize in e-commerce
- **jQuery dependencies** — Global dependencies create coupling throughout the frontend
- **Theme inheritance chain** — Blank → Luma → custom creates layers of override complexity

The fundamental issue: Luma was architected for server-side rendering with tight server-side coupling. Customizing it deeply often means fighting against the framework rather than working with it.

### Headless = Separate Frontend App + Magento as Pure API Backend

Headless Magento decouples the presentation layer from the commerce engine:

```
┌─────────────────┐         GraphQL/REST          ┌─────────────────┐
│   PWA/Hyvä      │ ←───────────────────────────→ │  Magento 2.4.8  │
│   Frontend      │        (API Only)            │  (Headless)     │
└─────────────────┘                               └─────────────────┘
     │                                                   │
     │         No checkout/phtml templates              │
     │                                                   │
     └───────────────────────────────────────────────────┘
                 API-driven only
```

**Benefits:**
- Frontend team works independently from backend
- Multiple frontends (web, mobile app, kiosk) share the same API
- Frontend technology choices can evolve faster than Magento releases
- Performance optimizations are fully under your control

**Challenges:**
- GraphQL becomes your only data contract
- SEO requires server-side rendering (SSR) or pre-rendering
- Checkout must be built from scratch (or use provided solutions)
- You lose some built-in Magento layout conventions

### The Three Real Options

| Approach | Technology | Maturity | Complexity |
|----------|------------|----------|------------|
| **Luma (Legacy)** | KnockoutJS + XML | Deprecated | Medium |
| **PWA Studio** | React + GraphQL | Magento-supported | High |
| **Hyvä** | React + Tailwind CSS | Third-party ecosystem | Medium |

PWA Studio is Magento's official headless direction. Hyvä is a third-party theme that provides React rendering while maintaining closer Magento integration than full PWA Studio. Both eliminate KnockoutJS entirely.

---

## 2. Magento PWA Studio (Venia)

### Project Structure: `pwa-studio/` Packages

PWA Studio is a monorepo containing multiple packages. The key packages:

```
pwa-studio/
├── packages/
│   ├── venia-ui/              # Venia storefront UI components
│   ├── peregrine/             # Component library + Talon hooks
│   ├── buildpack/             # Build tooling and configuration
│   ├── pagebuilder/           # PageBuilder integration for PWA
│   └── upward-spec/          # UPWARD server specification
└── packages/venia-concept/   # The reference storefront (Venia)
```

### Venia — The Reference PWA Storefront

**Venia** is the name of both the reference PWA storefront and the original package. It demonstrates PWA Studio capabilities:

- Full React-based storefront
- Service worker for offline support
- Lazy loading with intersection observer
- Cart, checkout, and customer account pages
- Product listing, detail, and search pages

Venia is production-ready as a reference but most teams build custom PWA storefronts inspired by Venia's patterns.

### Peregrine — Component Library and State Management

`@magento/peregrine` provides:

**Components** (`src/components/`):
- Reusable UI components (Button, Modal, ProductImage, Price)
- Composable design system built on Talon patterns

**Talon Hooks** (`src/talons/`):
- `useQuery` — GraphQL data fetching with loading/error states
- `useMutation` — GraphQL mutations with loading/error states
- `useCard` — Payment card handling
- `usePagination` — Paginated list state management
- `useScroll` — Scroll position and infinite scroll

**State Management**:
- Redux store for global state (cart, customer, UI)
- Local component state for UI-specific concerns
- Talon hooks provide the data layer for Redux

### Buildpack — Build Tooling

`@magento/buildpack` provides:

- **Webpack configuration** — Optimized for React and GraphQL
- **Environment configuration** — `MAGENTO_BACKEND_URL`, `BRANCH_NAME`, etc.
- **SSL certificate generation** — For local development HTTPS
- **Asset transformation** — Image optimization, CSS compilation
- **Module federation** — Allows loading Venia UI components into your custom PWA

The entry point is `packaging/createCustomOrigin.js` which sets up your custom domain with SSL.

### `packages/venia-ui/` — React Component Overrides vs `local-intercept.js`

Venia UI components are overridden using the `@magento/pwa-special-venues` pattern:

**Local Component Override** (recommended):
```javascript
// local-intercept.js
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetsOf = targets.of('@magento/venia-ui');

    // Override a component
    targetsOf.richTextRender.from(richText => {
        return richText.concat({
            component: 'src/targets/RichText',
            # path to your override
        });
    });

    // Add a static route
    targetsOf.intercept.add("route", (route) => {
        route.push({
            name: 'CustomPage',
            pattern: '/custom-page',
            path: 'src/components/CustomPage/target'
        });
    });
};
```

**File Override** approach:
```javascript
// overrides mapping in buildpack config
{
    "overrides": {
        "@magento/venia-ui": {
            "ProductImage": "src/components/ProductImageOverride"
        }
    }
}
```

### `packages/peregrine/` — Talon Hooks

Talon hooks follow a consistent pattern — each hook handles one GraphQL operation:

```javascript
// useQuery pattern
const { data, loading, error } = useQuery(GET_PRODUCT_DETAIL, {
    variables: { urlKey: productUrl }
});

// useMutation pattern
const [addToCart, { loading }] = useMutation(ADD_TO_CART_MUTATION);

// useCard pattern (payment)
const { cardData, validateCard } = useCard({
    onError: handleCardError
});
```

**Talon conventions**:
- Hooks return `{ data, loading, error }` tuple
- Loading state is always provided for optimistic UI
- Error boundaries catch GraphQL errors gracefully
- Variables are passed as second argument, options as third

### The Talon.js Pattern: Hooks for Every GraphQL Operation

Talon.js enforces a consistent pattern where every GraphQL operation has a corresponding hook:

| GraphQL Operation | Talon Hook |
|-------------------|------------|
| Query product | `useProductDetail` |
| Query cart | `useCartContext` |
| Mutation add to cart | `useAddToCart` |
| Mutation update quantity | `useQuantity` |
| Query customer | `useCustomer` |
| Query product search | `useProductSearch` |

This approach:
- **Encapsulates** GraphQL logic in reusable hooks
- **Separates** data fetching from presentation
- **Enables** easy mocking for testing
- **Maintains** consistent loading/error handling

### UPWARD Spec: `upward.yml` Defining the Node.js Server Response Strategy

UPWARD (Unified Progressive Web App Response Definition) defines how the Node.js server responds to requests:

```yaml
# upward.yml
services:
  venia:
    baseUrl: env.MAGENTO_BACKEND_URL
    port: 4000

response:
  engine: '@magento/venia-ui/server'

# Example: redirect to Magento for API calls
api:
  resolver: './server/APIResolver'

statusRoutes:
  200: render
  404: notFound
  500: internalError
```

**Key concepts**:
- `services` — Backend Magento connection
- `response` — How to render HTML (React server render)
- `api` resolver — How to handle GraphQL/API requests
- `statusRoutes` — HTTP status code mapping

For custom PWA projects, you typically extend Venia's UPWARD configuration rather than writing from scratch.

---

## 3. GraphQL-Only Data Layer

### PWA Studio Uses GraphQL Exclusively (No REST)

PWA Studio **only** communicates with Magento via GraphQL. REST is not used in the frontend:

```
PWA Frontend ←─────── GraphQL ────────→ Magento GraphQL Endpoint
                                  │
                                  └── No REST calls from PWA
```

**Why GraphQL?**
- Fetch exactly the data needed (no over-fetching)
- Single endpoint for all data
- Strong typing with schema
- Built-in support for real-time updates (subscriptions)

### `src/Peregrine/talons/` — All Data-Fetching Hooks

```
src/Peregrine/talons/
├── Cart/
│   ├── useCartPage.js
│   ├── useProductDetail.js
│   └── useAddToCart.js
├── Checkout/
│   ├── useCheckout.js
│   └── useBillingAddress.js
├── Wishlist/
│   └── useWishlist.js
└── General/
    ├── useQuery.js       # Generic query hook
    └── useMutation.js    # Generic mutation hook
```

Each Talon hook is self-documenting and exports its expected variables and return types.

### `src/queries/` — `.gql` Query Files

GraphQL queries are stored as `.gql` files:

```graphql
# src/queries/getProductDetail.gql
query getProductDetail($urlKey: String!) {
    products(filter: { url_key: { eq: $urlKey } }) {
        items {
            id
            name
            price_range {
                minimum_price {
                    final_price {
                        value
                        currency
                    }
                }
            }
            media_gallery {
                url
                label
            }
            description {
                html
            }
        }
    }
}
```

### Target Resolvers via GraphQL Fragments

Fragments allow reusable field selections:

```graphql
fragment PriceFragment on ProductInterface {
    price_range {
        minimum_price {
            final_price {
                value
                currency
            }
            regular_price {
                value
            }
        }
    }
}

query getProductWithPrice($urlKey: String!) {
    products(filter: { url_key: { eq: $urlKey } }) {
        items {
            id
            name
            ...PriceFragment
        }
    }
}
```

### How to Add a Custom GraphQL Query to PWA Studio

**Step 1: Define the GraphQL query in Magento** (server-side):
```graphql
# Magento's etc/schema.graphqls
type Query {
    myCustomData: MyCustomData @resolver(class: "Vendor\\Module\\Model\\Resolver\\CustomData")
}
```

**Step 2: Create a `.gql` file in your PWA project**:
```graphql
# src/queries/getCustomData.gql
query getCustomData {
    myCustomData {
        field1
        field2
    }
}
```

**Step 3: Create a Talon hook**:
```javascript
// src/talons/Custom/useCustomData.js
import { useQuery } from '@magento/peregrine/lib/talons/useQuery';
import getCustomDataQuery from 'src/queries/getCustomData.gql';

export const useCustomData = (options = {}) => {
    return useQuery(getCustomDataQuery, {
        fetchPolicy: 'cache-first',
        nextFetchPolicy: 'cache-and-network',
        ...options
    });
};
```

**Step 4: Use in a component**:
```javascript
import { useCustomData } from 'src/talons/Custom/useCustomData';

const MyComponent = () => {
    const { data, loading } = useCustomData();

    if (loading) return <LoadingIndicator />;
    return <div>{data?.myCustomData?.field1}</div>;
};
```

---

## 4. Creating a Custom PWA Page (Example)

### Intercepting Routes via `routes.js`

Routes are defined in `src/routes.js`:

```javascript
import React from 'react';
import { Route } from '@magento/venia-ui/lib/components/Router';
import MyCustomPage from 'src/components/MyCustomPage';

export const routes = [
    {
        path: '/custom-deals',
        component: MyCustomPage
    }
];
```

Or intercept existing routes via `local-intercept.js`:
```javascript
module.exports = function localIntercept(targets) {
    targets.of('@magento/venia-ui').routes.intercept((routes) => {
        routes.push({
            path: '/exclusive-deals',
            name: 'ExclusiveDeals',
            component: 'src/components/ExclusiveDealsPage'
        });
    });
};
```

### Creating a React Component

```jsx
// src/components/ExclusiveDealsPage/exclusiveDealsPage.js
import React from 'react';
import { useExclusiveDeals } from 'src/talons/Deals/useExclusiveDeals';
import ProductCard from 'src/components/ProductCard';

const ExclusiveDealsPage = () => {
    const { data, loading, error } = useExclusiveDeals();

    if (loading) return <LoadingIndicator />;
    if (error) return <ErrorMessage onRetry={refetch} />;

    return (
        <main className="exclusive-deals-page">
            <h1>Exclusive Deals</h1>
            <div className="product-grid">
                {data?.products?.items?.map(product => (
                    <ProductCard key={product.id} product={product} />
                ))}
            </div>
        </main>
    );
};

export default ExclusiveDealsPage;
```

### Writing a GraphQL Query

```graphql
# src/queries/getExclusiveDeals.gql
query getExclusiveDeals {
    products(
        filter: {
            category_id: { eq: "15" }
        }
        pageSize: 20
    ) {
        items {
            id
            name
            url_key
            price_range {
                minimum_price {
                    final_price {
                        value
                        currency
                    }
                }
            }
            media_gallery {
                url
            }
        }
        total_count
    }
}
```

### Connecting via Talon Hook

```javascript
// src/talons/Deals/useExclusiveDeals.js
import { useQuery } from '@magento/peregrine/lib/talons/useQuery';
import getExclusiveDealsQuery from 'src/queries/getExclusiveDeals.gql';

export const useExclusiveDeals = (options = {}) => {
    return useQuery(getExclusiveDealsQuery, {
        skip: false,
        ...options
    });
};
```

### Testing with `yarn run watch`

```bash
# Start development server with hot reload
yarn run watch

# Or for custom origin
yarn buildpack create-custom-origin .

# Run in production mode (for testing)
yarn run stage:dev
```

The watch command:
- Starts Webpack dev server
- Enables hot module replacement
- Sets up source maps for debugging
- Watches for file changes and rebuilds

---

## 5. Hyvä Themes

### What Hyvä Is: React-Based Alternative to Luma Using Tailwind CSS

Hyvä is a complete React-based theme for Magento 2 that:

- **Replaces KnockoutJS** with React
- **Replaces Luma XML layouts** with Tailwind CSS
- **Maintains Magento's built-in React rendering** (no separate Node.js server)
- **Provides ~70+ compatibility modules** for popular extensions

Hyvä's architecture is closer to traditional Magento theming than PWA Studio, but with modern React instead of KnockoutJS.

### Hyvä Themes (`Swissup/Aheadscale/Hyva_*`)

The Hyvä ecosystem includes:

| Package | Purpose |
|---------|---------|
| `Swissup/Aheadscale/Hyva_Theme` | Base theme |
| `Swissup/Aheadscale/Hyva_Grid` | Product grid system |
| `Swissup/Aheadscale/Hyva_Wishlist` | Wishlist functionality |
| `Swissup/Aheadscale/Hyva_Review` | Product reviews |
| `Swissup/Aheadscale/Hyva_Checkout` | React checkout |

Plus 70+ additional compatibility modules for checkout, CMS, search, and more.

### How Hyvä Works: React Components via Magento's Built-in React Rendering

Magento 2.3+ includes built-in React rendering support via `Magento_ReactContent`:

```
Hyvä Flow:
1. Customer requests page → Magento handles request
2. Hyvä XML layout triggers React component rendering
3. React components render via Magento_ReactContent
4. HTML + minimal JS sent to browser (SSR)
```

This differs from PWA Studio which requires a separate Node.js server. Hyvä runs on standard PHP hosting:

```xml
<!-- Hyvä layout XML -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_layout.xsd">
    <update handle="hyva_react"/>
    <referenceBlock name="content">
        <block class="Hyva\\React\\Block\\Component" name="product.list"/>
    </referenceBlock>
</page>
```

### `@hyva/react-checkout` — React Checkout Replacing KnockoutJS

Hyvä's checkout is provided by `@hyva/react-checkout`:

```javascript
// In Hyvä-compatible checkout module
import { CheckoutProvider } from '@hyva/react-checkout';

const MyCheckout = () => (
    <CheckoutProvider>
        <CartSummary />
        <ShippingStep />
        <PaymentStep />
        <OrderReview />
    </CheckoutProvider>
);
```

This replaces Luma's complex KnockoutJS checkout with clean React components.

### Compatibility Matrix

| Extension Type | Hyvä Support |
|----------------|--------------|
| PageFly / MetaLayouts | Works (page builders) |
| Amasty extensions | Partial (check module list) |
| Aheadworks extensions | Hyvä-specific versions available |
| Mirasvit extensions | Hyvä-compatible versions available |
| Custom extensions | May need Knockout removal work |

**Before choosing Hyvä**, verify all critical extensions have Hyvä compatibility.

### When Hyvä Is the Right Choice

**Choose Hyvä when:**
- Your team knows React (not GraphQL/Node.js)
- You want SSR performance without managing a Node.js server
- You need faster time-to-market than PWA Studio allows
- You don't require full PWA capabilities (offline, push notifications)
- You have extensions that already support Hyvä

**Don't choose Hyvä when:**
- You need full PWA features (service worker, offline)
- You're building a marketplace with complex seller dashboards
- Your team knows React Native for mobile

---

## 6. PWA Studio vs Hyvä vs Luma — Decision Matrix

| Criteria | Luma | PWA Studio | Hyvä |
|----------|------|------------|------|
| **Tech Stack** | KnockoutJS + XML | React + GraphQL | React + Tailwind |
| **Architecture** | Server-rendered | Separate Node.js server | Server-rendered via Magento |
| **Performance** | Slow | Fast (PWA + lazy loading) | Fast (SSR) |
| **Dev Complexity** | Medium | **High** | Medium |
| **SEO** | Good | Excellent | Excellent |
| **Offline/PWA** | No | Yes | No (SSR only) |
| **Checkout** | Built-in (Knockout) | Custom built | `@hyva/react-checkout` |
| **GraphQL** | Optional | Required | Optional |
| **Node.js Server** | Not required | **Required** | Not required |
| **Magento Version** | Any | 2.4+ only | 2.4+ only |
| **Maintenance Burden** | Medium (deprecated) | High | Medium |
| **Theme Inheritance** | XML cascading | Component overrides | Tailwind + layout XML |
| **Hot Module Reload** | No | Yes (dev mode) | No (standard Magento) |
| **Build Time** | Fast | Slow (Webpack) | Fast |
| **Extension Compatibility** | Full | Partial (GraphQL required) | Partial (requires Hyvä modules) |

### When Each Approach Excels

**Luma** — Only for teams maintaining existing Luma themes or needing maximum extension compatibility without custom development.

**PWA Studio** — Teams building marketplaces, needing offline capabilities, or planning native mobile apps alongside the web storefront.

**Hyvä** — Teams transitioning from Luma who want React without the complexity of managing a separate Node.js infrastructure.

---

## 7. Which to Choose

### Choose PWA Studio When:

Your team has React experience and you need:
- **Full PWA capabilities** — offline browsing, push notifications, installable
- **Marketplace architecture** — seller dashboards, multi-vendor product management
- **Mobile companion** — PWA Studio patterns translate well to React Native
- **Complex interactions** — drag-and-drop page builders, real-time collaboration
- **GraphQL-first mindset** — team is comfortable with GraphQL API design

**Team size consideration**: PWA Studio requires dedicated frontend and possibly DevOps resources for the Node.js server.

### Choose Hyvä When:

Your team has React experience and you need:
- **Faster time-to-market** — less setup than full PWA Studio
- **SSR performance** — Google SEO scores without managing Node.js
- **Existing extension compatibility** — modules you need already have Hyvä versions
- **Standard hosting** — no Node.js server maintenance
- **Familiar Magento patterns** — XML layouts still work, just with React components

**Team size consideration**: Hyvä works well with one or two frontend developers who know React basics.

### Choose Neither (Standard Luma + BFF) When:

- You need simple headless delivery (CMS content, catalog data)
- Your frontend team is not React-fluent
- You have budget for custom Knockout work
- You need maximum extension compatibility immediately

**Recommendation**: Implement a Backend-for-Frontend (BFF) pattern with Luma:
- Magento provides GraphQL API
- Lightweight BFF (Node.js/Go) transforms data
- Any frontend (Vue, React, vanilla JS) consumes the BFF

### Migration Path: Luma → Hyvä → PWA Studio

```
Luma (current)
    ↓ (replace checkout first)
Hyvä (React + Tailwind)
    ↓ (add PWA capabilities)
PWA Studio (full headless)
```

Most teams migrate incrementally: first Hyvä (simpler), then evaluate PWA Studio if full PWA features become necessary.

### Implementation Checklist

**For PWA Studio:**
- [ ] Set up Node.js 18+ server environment
- [ ] Clone and configure PWA Studio packages
- [ ] Define GraphQL schema extension for custom data
- [ ] Create Talon hooks for all custom queries
- [ ] Build component tree using Venia patterns
- [ ] Configure UPWARD for production deployment
- [ ] Set up CI/CD for the Node.js application

**For Hyvä:**
- [ ] Install base Hyvä theme package
- [ ] Configure Tailwind build system
- [ ] Install required Hyvä compatibility modules
- [ ] Replace Luma checkout with `@hyva/react-checkout`
- [ ] Convert Knockout templates to React components
- [ ] Test extension compatibility
- [ ] Enable Magento's built-in React rendering

---

## See Also

- [GraphQL Schema Extension](../_supplemental/11-graphql-rest-api.md) — Extending the GraphQL schema PWA Studio consumes
- [REST/GraphQL API Development](../07-api-development/11-graphql-rest-api.md) — General API development patterns
- [Magento UPWARD Specification](https://developer.adobe.com/commerce/pwa-studio/guides/configure/upward/) — UPWARD server configuration