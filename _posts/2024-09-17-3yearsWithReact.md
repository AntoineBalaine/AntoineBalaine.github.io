---
layout: post
title: "Three years with React - my personal list of best practices"
date: 2024-09-17
categories: [Development, React]
tags: [react, typescript, best-practices, coding-standards]
description: "My personal best practices after three years of React/TypeScript development"
toc: true
---

- [Disclaimer:](#disclaimer)
- [Type your data, damn it!](#type-your-data-damn-it)
- [Use type predicates](#use-type-predicates)
- [Do not use the spread operator to deep copy](#do-not-use-the-spread-operator-to-deep-copy)
- [Global side effects should not trigger other global side effects](#global-side-effects-should-not-trigger-other-global-side-effects)
- [Don’t use mutable refs in lieu of state](#dont-use-mutable-refs-in-lieu-of-state)
- [Don’t re-export variables from global context, unless lower levels are forbidden from accessing it](#dont-re-export-variables-from-global-context-unless-lower-levels-are-forbidden-from-accessing-it)
- [Validate the content of data that’s entering the app.](#validate-the-content-of-data-thats-entering-the-app)
- [If your team’s not testing enough, it’s probably because they don’t know how to.](#if-your-teams-not-testing-enough-its-probably-because-they-dont-know-how-to)


A few thoughts, scattered along the way, about the dos and don’ts that have plagued the project to which I contributed for the past three years.

## Disclaimer:
Typing data is not for everyone.

There are indeed engineers who are capable of keeping track of where things are in the codebase, of who’s working on what, and who have such a wide view of things that they can afford to write code without guardrails.

Now, for the rest of us mortals, we need the text editor to tell us when we’re messing up. The bigger the project, the more important that becomes.

Also, I have seen some very competent and very _señor_ engineers (business-communication skills, great architecture design, deep understanding of technical limitations of the framework) totally fail the switch to typescript. Coming from a life of dynamic data, the «why» of the language completely eluded them. It did not fit their brain.

As a result, these were the engineers that most often resorted to using `any` all over the place, and made silly mistakes. This caused issues that constantly had to be cleaned up by their teammates. And by their teammates, I mean me. One of them used to joke: «so Anthony, whose butt are you going to have to wipe today?»

Dear reader, if you are about to endeavour a big typescript project, and given how many times I have had to be the code cleaning lady, please consider this list.

## Type your data, damn it!
Mis-typed data have been 70% of our bugs, right there. The business would have saved millions if we had been strict from the get go.

- `any` should be strictly forbidden, unless in case of absolute necessity.
- if by `any` you mean `string` or `number` or `boolean`, then your type is `string|number|boolean`, NOT `any`.
- if by `any` you mean `unknown`, then it’s `unknown` and you need to use a type predicate to validate the data.
- type-casting should be avoided in favor of type predicates and other strategies.

## Use type predicates
If some data’s `unknown`, check it:
```typescript
function isAxiosError<T>(response: unknown): response is AxiosError<T> {
  const hasAxiosError = (o: object): o is { isAxiosError: unknown } => "isAxiosError" in o;
  if (
    response !== null &&
    typeof response == "object" &&
    hasAxiosError(response) &&
    response.isAxiosError === true
  ) {
    return response.isAxiosError;
  } else return false;
}
```
This small check is _much_ safer than using type-casting. Consider this cast:
```typescript
const t: string = "hello";
transformNumber(t as number);
```
This is just a white lie given to the type-checker. Inevitably, relying a lot on casts introduces inaccuracies in the system - which _promise_ to pile up until the code cleaning lady comes. Adding lies on top of lies does not make solid software, does it?
Here’s an easy solution:
```typescript
export const isNumber = function (datum: unknown ): datum is number{
  return typeof datum === "number";
}

const t: string = "hello";
isNumber(t) && transformNumber(t as number);
```

## Do not use the spread operator to deep copy
Doing `{...myDatum}` does not create a deep copy, it only copies down 1-level of nesting. This can become very problematic if you have a large top-level datum on which your whole component tree depends. If you’re only copying superficially, some of your deeper-nested components might fail to reflect the render update.

As a solution, using react requires using a cloner function.
Before you jump about creating a `recursiveClone()` helper, there’s lodash’s excellent `cloneDeep()`. Look into its implementation and you’ll understand why I recommend it over an in-house solution.

## Global side effects should not trigger other global side effects
So much maintenance pain came from this.
Say I have datum B which depends on datum A, which depends on an API call. Since the state updates are asynchronous, you choose to wait for A’s update, and then update B via side effect.
```typescript
async ()=>{
  const res = await axios.get<typeof A>();
  setA(res.data);
}

useEffect(()=>{
  setB(transformA(A));
},[A])
```
That’s ok for a todo app - not ok for professional-grade software.

If you do this, the more your app grows, the harder it becomes to debug and maintain. The flow of data becomes scattered across many side effects, which makes it difficult to track bugs. It becomes increasingly hard to figure out which part is causing the issue in the effects chain.
Something much simpler:
```typescript
async ()=>{
  const res = await axios.get<typeof A>();
  setA(res.data);
  setB(transformA(res.data));
}
```
## Don’t use mutable refs in lieu of state
The async nature of `useState` causes some difficulties - that’s functional-prog’s influence on React’s design.
Immutable data and strict rules don’t always fit your mental model, and you might tend to revert back to using mutable-style data. Unless you plan to `useMemoize` a lot (data table components are a notable illustration of this need), check this again:
```typescript
async ()=>{
  const res = await axios.get<TypeA>();
  const a = transform1(res.data)
  setA(cloneDeep(a));
  const b = transform2(a);
  setB(cloneDeep(b));
}
```
- the data stays immutable,
- the transforms are pretty clear,
- the async updates are not a problem.

## Don’t re-export variables from global context, unless lower levels are forbidden from accessing it
Minor nit-pick here: if re-exports are only implemented in certain parts of the codebase, that could be a sign of lack of cohesiveness amongst teams.

Whether this means that you’re surrounded by sweet dreamers or sitting on a volcano - keep a close eye.
If your data is flowing down multiple contexts which each operate their own transforms, your control flow might get weird quickly.

## Validate the content of data that’s entering the app.
1. Why are you not typing the contents of API responses?
```typescript
axiosInstance.get<ResponseType|ErrorType>();
```
What’s the use of TS if entry points into the application are not typed?

2. Why are you not validating the contents of API responses?
```typescript
const res = await axiosInstance.get<ResponseType|ErrorType>();
if (isAxiosError(res) || !res.status == 200 || !isValidResponseType<ResponseType>(res)) throw;
```
Same thing: what’s the point of typing your entry points if you’re not checking that they honor their contract?

## If your team’s not testing enough, it’s probably because they don’t know how to.
Testing is always the dev’s poor relative.
It took our leads years to realize that our competent react engineers did not know _what_ to test. Years…

If your team lacks testing and it’s not for lack of wanting, then you might be facing a skill gap in your devs.

I suspect that’s more common than not - management teams might throw JS devs at a TS project without affording them any kind of training. This makes for gaps in a team’s knowledge; and without strong leads who are capable of assessing them, you might end up building on top of shaky work.

The knee jerk reaction to this might be to implement code coverage requirements on PRs, but this also comes with the downside that it encourages devs to test implementation instead of functionality - yet another bane for maintainability…
