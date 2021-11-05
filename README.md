# How to Type our GraphQL queries (WP GraphQL + NextJS, part 2)

This is the second part of the '_Create a Next.js static site with Wordpress and GraphQL_' tutorial. In the [first part](https://github.com/HynekS/wp-next-graphql-tutorial), we have learned how to connect WordPress (as a 'backend') and Next.js (as a 'frontend') using GraphQL queries and how to export our app as a static site.

## Prerequisites

The prerequisites are same as of the [first part](https://github.com/HynekS/wp-next-graphql-tutorial). If you've already finished it, you can [skip that part](#install-and-initialize-graphql-code-generator). If you don't want to follow the first tutorial, you can clone the github repo as a starting point:

```bash
git clone --recurse-submodules https://github.com/HynekS/wp-next-graphql-tutorial/
```

You will also need to set up a few more things, though:

### Wordpress

You will need to create a database for WordPress. In In the `/bedrock` directory, you'll need to rename the `.env.example` file to`.env`. In the `.env` file, you'll need to update your database credentials and the base url of the site. For installing all dependencies, you'll need to run `composer install --no-scripts` command. You'll need to activate the '**WP GraphQL**' plugin and flush permalinks (Settings -> Permalinks, Save Changes). It's probably a good idea to create a few posts using **Faker Press** plugin (or manually, if you feel like it). For more details, check the [first part](https://github.com/HynekS/wp-next-graphql-tutorial).

### Next.js

In the `/nextjs` folder, run `yarn install`. When all the dependencies are installed (and PHP/MySQL servers for WordPress are up and running) start NextJS by `yarn dev` command.

## Install and initialize the GraphQL Code generator

The GraphQL Code generator install is requiring a couple of steps. Let's go through it:

### Instal the core modules

First, we need to install the core dependencies:

```bash
yarn add graphql
yarn add -D @graphql-codegen/cli
```

### Initialize the code generator

Next, we need to initialize the code generator, which will give us a config file:

```bash
 yarn graphql-codegen init
```

After that, the `init` script will ask us several questions about our app. Let's use the answers as listed below:

<pre>
Welcome to GraphQL Code Generator!
Answer few questions and we will setup everything for you.

? What type of application are you building? <em>Application built with React</em>
? Where is your schema?: (path or url) <em>http://127.0.0.1/wp/graphql</em>
? Where are your operations and fragments?: <em>lib/api.(js|ts)</em>
? Pick plugins: <em>TypeScript (required by other typescript plugins), TypeScript Operations (operations and fragments)</em>
? Where to write the output: <em>generated/graphql.ts</em>
? Do you want to generate an introspection file? <em>Yes</em>
? How to name the config file? <em>codegen.yml</em>
? What script in package.json should run the codegen? <em>codegen</em>
</pre>

The code generator will install some additional dependencies and create a new file in our root directory, `codegen.yml`. It is a config file. If we open it, it should look roughly like that:

```yml
# codegen.yml

overwrite: true
schema: "http://127.0.0.1/wp/graphql"
documents: "lib/api.(js|ts)"
generates:
  generated/graphql.ts:
    plugins:
      - "typescript"
      - "typescript-operations"
  ./graphql.schema.json:
    plugins:
      - "introspection"
```

## Turn GraphQL introspection ON in WP GraphQL plugin settings

There is one important thing we need to update in our WordPress settings. In the bottom of the _WP GraphQL_ plugin settings page, there is a 'Public Introspection Enabled' checkbox. [For security reasons, it is disabled by default](https://www.wpgraphql.com/docs/security/):

> One feature of GraphQL is Schema Introspection, which means the GraphQL Schema itself can be queried. This is a feature used by tools such as GraphiQL and others.
> It's possible that exposing the Schema publicly (in some cases) can leak information about the system that's not intended to be known publicly.

But we need to enable it – codegen need to know our Schema to generate the types. On our dev environment, there is probably not much to worry about (according to this PR comment, the introspection [should be disabled by default](https://github.com/wp-graphql/wp-graphql/pull/1490#issue-715306162) on _staging_ and _production_ environments). Let's turn the introspection **ON** and save the changes.

![Image of a settings menu with enhanced introspection checkbox](assets/wp-graphql-tutorial-pt2-enable-introspection.png)

## Add the GraphQL Tag Plucks

If we try to run the `yarn run codegen` script, we'll still get an error: `Unable to find any GraphQL type definitions for the following pointers`. That's because the code generator is trying to find our GraphQL queries to create a type definitions – but it can't find any. All we are pointing it to (the 'documents' field in our config) is a file that has a few string arguments recognizable as GraphQL queries by a human – but the program is not smart enough (yet). We need to give it a hand.

There are some rather sophisticated solutions to that, like using a custom Webpack loaders for `.graphql` files, but we'll keep it simple ([stupid too](https://en.wikipedia.org/wiki/KISS_principle), maybe). We'll give the code generator some hints. These are [multiple patterns](https://www.graphql-code-generator.com/docs/getting-started/documents-field#graphql-tag-pluck) available – we'll choose the `/* GraphQL */` comment, which we'll put right before the opening backtick of the query string:

```TypeScript
// lib/api.js

export async function getLatestPosts() {
  const data = await fetchAPI(/* GraphQL */ `
    query LatestPosts {
      posts {
        nodes {
          title
          slug
        }
      }
    }
  `);
  return data?.posts;
}
```

(I've noticed that after inserting the `/* GraphQL */` hint, the query string was recognized by VS Code as a proper GraphQL query and changes its color. But it might depend on the actual editor settings.)

This method of tagging strings that are actually GraphQL queries is being referred to as a [GraphQL Tag Pluck](https://www.graphql-code-generator.com/docs/getting-started/documents-field#graphql-tag-pluck) – hence the header.

## Update the .gitignore file

When running the codegen, we will generate some files that we probably don't want to be included in our version control. We'll prevent that by adding these  lines to `.gitignore`:

```.gitignore
// ...

# generated schema and types
graphql.schema.json
generated/
```

## Run the codegen script

Great! Everything should be set up now. Let's try our `codegen` script.

```bash
yarn run codegen
```

The very first run may take a few seconds. Hopefully, the script resolved successfully and we cas see two more (rather big) files – `graphql.schema.json` in our root directory, containing our GraphQL schema, and `/generated/graphql.ts`, containing the derived types. If we open the latter and scroll to the bottom of the file, we should see the types for  queries we defined in `lib/api.js` file:

```typescript
// generated/graphql.ts

//...

export type LatestPostsQueryVariables = Exact<{ [key: string]: never }>;

export type LatestPostsQuery = {
  __typename?: "RootQuery";
  posts?:
    | {
        __typename?: "RootQueryToPostConnection";
        nodes?:
          | Array<
              | {
                  __typename?: "Post";
                  title?: string | null | undefined;
                  slug?: string | null | undefined;
                }
              | null
              | undefined
            >
          | null
          | undefined;
      }
    | null
    | undefined;
};

export type AllPostsWithSlugQueryVariables = Exact<{ [key: string]: never }>;

export type AllPostsWithSlugQuery = {
  __typename?: "RootQuery";
  posts?:
    | {
        __typename?: "RootQueryToPostConnection";
        nodes?:
          | Array<
              | { __typename?: "Post"; slug?: string | null | undefined }
              | null
              | undefined
            >
          | null
          | undefined;
      }
    | null
    | undefined;
};

export type PostQueryVariables = Exact<{
  id: Scalars["ID"];
}>;

export type PostQuery = {
  __typename?: "RootQuery";
  post?:
    | {
        __typename?: "Post";
        content?: string | null | undefined;
        date?: string | null | undefined;
        title?: string | null | undefined;
      }
    | null
    | undefined;
};
```

## Utilize the types in our api

For simplicity sake, we left the `/lib/api.js` as a plain javascript file tn the first part ot the tutorial. Now, it's the time to make a step forward and convert it to TypeScript – simply by changing its extension: `mv lib/api.js lib/api.ts`. The compiler may have some complaints (about parameters without types) – don't worry, we'll fix that soon.

### Import and arrange types for our queries

First, let's import our generated types. Also, let's separate them to two basic categories: the `Query` types and query variables, which we put inside the general `Options` type.

```typescript
// lib/api.ts

import type {
  LatestPostsQueryVariables,
  LatestPostsQuery,
  AllPostsWithSlugQueryVariables,
  AllPostsWithSlugQuery,
  PostQueryVariables,
  PostQuery,
} from "../generated/graphql";

type Query = LatestPostsQuery | AllPostsWithSlugQuery | PostQuery;

type Options = {
  variables?:
    | LatestPostsQueryVariables
    | AllPostsWithSlugQueryVariables
    | PostQueryVariables;
};

const API_URL = "http://127.0.0.1/wp/graphql";

export async function fetchAPI(query, { variables }) {
  // ...
}

// ...
```

### Add types the fetchApi function

Now, let's type out `fetchAPI` function. It will accept a generic type `T` with the constraint that it is a `T` of the `Query` type: `<T extends Query>`. The functions returns that `T` type wrapped in a Promise: `Promise<T>`.

Also, let's add types to its parameters: the `query` is simply a `string`, the options object (now anonymous because of the destructuring) will take the `Options` type. We'll also give it a default argument – an empty object `{}`.

```typescript
// lib/api.ts

import type {
  LatestPostsQueryVariables,
  LatestPostsQuery,
  AllPostsWithSlugQueryVariables,
  AllPostsWithSlugQuery,
  PostQueryVariables,
  PostQuery,
} from "../generated/graphql";

type Query = LatestPostsQuery | AllPostsWithSlugQuery | PostQuery;

type Options = {
  variables?:
    | LatestPostsQueryVariables
    | AllPostsWithSlugQueryVariables
    | PostQueryVariables;
};

const API_URL = "http://127.0.0.1/wp/graphql";

export async function fetchAPI<T extends Query>(
  query: string,
  { variables }: Options = {}
): Promise<T> {
  const headers = { "Content-Type": "application/json" };

  const res = await fetch(API_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });

  const json = await res.json();

  if (json.errors) {
    console.log(json.errors);
    console.log("error details", query, variables);
    throw new Error("Failed to fetch API");
  }
  return json.data;
}
```

### Add types for our queries

Our `fetchAPI` function accepts a generic type `<T>`. In the specialized querying functions, we will use replace the generic type with one of the concrete types we imported beforehand.

The `getPost` function also accepts a query variables object as argument. Let's give it the imported `PostQueryVariables` type.

Our `api.ts` file should look like that (hopefully, no TypeScript complaints):

```typescript
// lib/api.ts

import type {
  LatestPostsQueryVariables,
  LatestPostsQuery,
  AllPostsWithSlugQueryVariables,
  AllPostsWithSlugQuery,
  PostQueryVariables,
  PostQuery,
} from "../generated/graphql";

type Query = LatestPostsQuery | AllPostsWithSlugQuery | PostQuery;

type Options = {
  variables?:
    | LatestPostsQueryVariables
    | AllPostsWithSlugQueryVariables
    | PostQueryVariables;
};

const API_URL = "http://127.0.0.1/wp/graphql";

export async function fetchAPI<T extends Query>(
  query: string,
  { variables }: Options = {}
): Promise<T> {
  const headers = { "Content-Type": "application/json" };

  const res = await fetch(API_URL, {
    method: "POST",
    headers,
    body: JSON.stringify({ query, variables }),
  });

  const json = await res.json();

  if (json.errors) {
    console.log(json.errors);
    console.log("error details", query, variables);
    throw new Error("Failed to fetch API");
  }
  return json.data;
}

export async function getLatestPosts() {
  const data = await fetchAPI<LatestPostsQuery>(/* GraphQL */ `
    query LatestPosts {
      posts {
        nodes {
          title
          slug
        }
      }
    }
  `);
  return data?.posts;
}

export async function getAllPostsWithSlug() {
  const data: AllPostsWithSlugQuery =
    await fetchAPI<AllPostsWithSlugQuery>(/* GraphQL */ `
      query AllPostsWithSlug {
        posts(first: 100) {
          nodes {
            slug
          }
        }
      }
    `);
  return data?.posts;
}

export async function getPost(slug: PostQueryVariables["id"]) {
  const data: PostQuery = await fetchAPI<PostQuery>(
    /* GraphQL */ `
      query Post($id: ID!) {
        post(id: $id, idType: SLUG) {
          content
          date
          title
        }
      }
    `,
    {
      variables: {
        id: slug,
      },
    }
  );

  return data?.post;
}
```

## Utilize the types in our templates

Great! We already have typed the data we're querying in our api. But the page templates doesn't know the types yet. We need to do a one last step.

## Import and apply our types

Let's import `InferGetStaticPropsType` type utility from "next" module. Then, we'll simply use that utility to infer the return type of the `getStaticProps` function:

```typescript
// pages/index.tsx

import Link from "next/link";

import type { InferGetStaticPropsType } from "next";

import { getLatestPosts } from "./../lib/api";

const Home = ({
  latestPosts,
}: InferGetStaticPropsType<typeof getStaticProps>) => {
  const { nodes } = latestPosts;

  return (
    <div>
      <h1>Our WordPress/Next app Home Page</h1>
      <ul>
        {nodes.map((node) => (
          <div key={node.slug}>
            <Link href={"posts/" + node.slug}>{node.title}</Link>
          </div>
        ))}
      </ul>
    </div>
  );
};

export async function getStaticProps() {
  const latestPosts = await getLatestPosts();

  return {
    props: {
      latestPosts,
    },
  };
}

export default Home;
```

Now, we should be able to see the inferred type when hovering over the `LatestPosts` prop:

![Image of the type 'tooltip' appearing on variable hover in VS Code](assets/wp-graphql-tutorial-pt2-inferred-type.png)

### Update our component to make TypeScript happy

But TypeScript compiler is not happy! It is spawning wavy red underlines everywhere!

Don't panic. TypeScript in a strict mode (which is a default for NextJS v12) can be pretty harsh. The problem is that in our inferred `LatestPosts`, very little of the properties is guaranteed. Therefore, we need to update our code so it wont't crash at runtime with timeless `Uncaught TypeError: Cannot read property 'X' of undefined` (or null).

Since our component is simple, all we'll need is [nullish coalescence](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_operator) (`??`) and [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) (`?.`) operators:

```typescript
// pages/index.tsx

import Link from "next/link";

import type { InferGetStaticPropsType } from "next";

import { getLatestPosts } from "./../lib/api";

const Home = ({
  latestPosts,
}: InferGetStaticPropsType<typeof getStaticProps>) => {
  const { nodes } = latestPosts ?? {};

  return (
    <div>
      <h1>Our Wordpress/Next app Home Page</h1>
      <ul>
        {nodes?.map((node) => (
          <div key={node?.slug}>
            <Link href={"posts/" + node?.slug}>{node?.title}</Link>
          </div>
        ))}
      </ul>
    </div>
  );
};

export async function getStaticProps() {
  const latestPosts = await getLatestPosts();

  return {
    props: {
      latestPosts,
    },
  };
}

export default Home;
```

Now, TypeScript should be happy.

### The `[slug].tsx` template

In the `[slug].tsx` file, we will need to import one more utility type – `GetStaticPropsContext`. It will be used for the arguments passed in `getStaticProps` function. We're passing in the object literal (for simplicity – it is probably a good practice to extract that in a separate type) with the `slug` property of type `string`.

We are also probably getting a lot od TypeScript errors due to unsafe property chaining. Lets fix it. We will use the nullish coalescing and optional chaining as before:

```typescript
// pages/posts/[slug].tsx

import Link from "next/link";

import type { GetStaticPropsContext, InferGetStaticPropsType } from "next";

import { getAllPostsWithSlug, getPost } from "../../lib/api";

const Post = ({ post }: InferGetStaticPropsType<typeof getStaticProps>) => {
  let { title, date, content } = post ?? {};

  return (
    <div>
      <Link href="/">Back to Homepage</Link>
      <h1>{title}</h1>
      {date && <time dateTime={date}>{new Date(date).toDateString()}</time>}
      <div dangerouslySetInnerHTML={{ __html: content ?? "" }}></div>
    </div>
  );
};

export async function getStaticProps(
  context: GetStaticPropsContext<{ slug: string }>
) {
  const slug = String(context.params?.slug);
  const post = await getPost(slug);
  return {
    props: {
      post,
    },
  };
}

export async function getStaticPaths() {
  const allPostsWithSlug = await getAllPostsWithSlug();
  return {
    paths: allPostsWithSlug?.nodes?.map((node) => `/posts/${node?.slug}`) ?? [],
    fallback: false,
  };
}

export default Post;
```

Well, that's all for now. Hope you've learned something useful while reading this. You can check the final code in my github repo.

👍 Enjoy!
