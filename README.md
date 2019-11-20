# Nx 8.12: 10x Faster CI With New Distributed Caching

Nx 8.12 now supports distributed caching. This repository shows how it works.

## Intro Video

Before reading further, watch this 5-minute video.

<p align="center">
<a href="https://www.youtube.com/watch?v=w1-GiB74ddc&feature=emb_logo" target="_blank">
<img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/video-screenshot.png" width="800">
</a>
</p>

Want to enable distributed caching for your workspace? Go to [https://nx.app](https://nx.app).

## Example Workspace

This repo has two applications. Each app has 15 libraries, each of which consists of 30 components. The two applications also share code.

If we run `nx dep-graph`, we will see something like this:

<p align="center"><img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/graph.png" width="800"></p>

### CI Provider

This example will use CircleCI, but a very similar setup will work with Azure, Jenkins, GitLab, etc..

**To see CI runs click [here](https://circleci.com/gh/nrwl/nx-distributed-cache-example).**

The CircleCI configuration looks like this:

```yaml
jobs:
  build-master:
    executor: default
    steps:
      - setup
      - run: yarn nx affected --target=lint --base=origin/master~1 --parallel
      - run: yarn nx affected --target=test --base=origin/master~1 --parallel
      - run: yarn nx affected --target=build --base=origin/master~1 --parallel
  build-pr:
    executor: default
    steps:
      - setup
      - run: yarn nx affected --target=lint --base=origin/master --parallel
      - run: yarn nx affected --target=test --base=origin/master --parallel
      - run: yarn nx affected --target=build --base=origin/master --parallel
```

As you can see, we are utilizing Nx's 'affected', so we only rebuild and retest what is affected by our PR.

## Enabling Caching

To enable caching, we need to update `nx.json` (see https://nx.app for more info).

```json
 "tasksRunnerOptions": {
    "default": {
      "runner": "@nrwl/nx-cloud",
      "options": {
        "accessToken": "YTYzMThiN2UtYjgzYy00YzJiLTg5M2UtNzFjOGVkZWFiNDQ0fHJlYWQ=",
        "cacheableOperations": ["build", "lint", "test"]
      }
    }
 }
```

## CI

### First CI Run

If we submit a PR changing something fundamental about the workspace, everything will be affected. 

<p align="center"><img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/graph-one-affected.png" width="800"></p>

Because it's a large change, the CI will take about 15 minutes to complete. 

<p align="center"><img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/ci-1.png" width="800"></p>

Now imagine we missed something, and the CI failed. Without the cache, we would have to go to the CI, find what projects failed, and run only those projects locally. This can be tedious. But because we have the cache, we don't have to do that. We can simply run:

```
nx affected:test
```


We will see the following:

```
>  NX NOTE  Nx read the output from cache instead of running the command for the following projects:

  - react-lib14
  - react-lib4
  - react-lib5
  - react-lib1
  - react-lib3
  - ng-lib0
  - react-lib2
  - react-lib10
  - react-lib12
  - react-lib8
  - shared-utils
  - react-lib11
  - react-lib7
  - ng-lib2
  - ng-lib10
  - react-app
  - ng-lib4
  - ng-lib11
  - ng-lib1
  - react-lib9
  - ng-app
  - ng-lib3
  - ng-lib13
  - ng-lib9
  - react-lib13
  - react-lib6
  - ng-lib14
  - ng-lib8
  - ng-lib7
  - ng-lib5
  - ng-lib6
  - ng-lib12
  - react-lib0
```

Instead of running all the tests locally, Nx pulled what it can from the cache and only ran things that had failed on CI. This is what using caching is all about--nothing gets run twice. This alone is a huge quality of life improvement. 


### Second CI Run

Now imagine we pushed the code fix. Our PR still affects all the projects. If we didn't have the cache, it would have meant another 15-minute CI run. But because we do have the cache, it only takes 1.5 minutes, because most of the things get pulled from the cache.

<p align="center"><img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/ci-2.png" width="800"></p>

### Multiple CI Runs

In a real organization, most PRs run CI many times, often without any changes at all. For instance, most merges into master will likely to be fully cached, which means that an average master CI run should be under a minute:

<p align="center"><img src="https://raw.githubusercontent.com/nrwl/nx-distributed-cache-example/master/readme-assets/ci-3.png" width="800"></p>

## Greenfield Projects vs Real Projects

In greenfield projects enabling distributed cache can give a 10x CI speedup. Existing large projects with complicated CI setups probably won't become 10x faster, but in our experience, their average CI time still becomes 3-5 times faster. **This is without making any changes to the CI setup. Add 2 lines of code to `nx.json` and you will see the benefits the same day.**
 
In addition to enabling caching, `@nrwl/nx-cloud` collects some basic stats on how long different operations take, so you can see how much dev time you saved by enabling distributed caching.F

## Try It Locally   

* Clone this repo
* run `yarn`
* run `nx build react-app --prod`
* run `nx build ng-app --prod`
* run `nx run-many --target=test --all`
* run `nx run-many --target=lint --all`

All these commands will execute instantly. You will see the same output you'd have seen if you haven't had them cached. They will create the same files in the `dist` folder they would have created without the cache. The only difference between getting something from the cache and computing it is that using the cache is a lot faster.

**With Nx distributed caching, you never rebuild the same artifact twice.**

If your CI ran something, you can simply get the artifact with the terminal without running it again. If your team mate ran something on their machine, you can simply get their artifact with their output without running it again. 


## Questions

### Why use affected instead of just using the cache?

Affected and caching are used to solve the same problem: minimize the computation. But they do it differently, and the combination provides better results than one or the other.

The affected command looks at the before and after states of the workspaces and figures out what can be broken by a change. Because it knows the two states, it can deduce the nature of the change. For instance, this repository uses React and Angular. If a PR updates the version of React in the root `package.json`, Nx will know that only half of the projects in the workspace can be affected. It knows what was changed--the version of React was bumped up.

Caching simply looks at the current state of the workspace and the environment (e.g., version of Node) and check if somebody already ran the command against this state. Caching knows that something changed, but because there is no before and after states, it doesn't know the nature of the change. In other words, caching is a lot more conservative.

1. If we only use affected, the list of projects that will be retested is small, but if we test the PR twice, we will run all the tests twice.
2. If we only use caching, the list of projects that will be retested is larger, but if we test the PR twice, we will only run tests the first time.

**Using both allows us to get the best of both worlds. The list of affected projects is as small as it can be, and we never run anything twice.**

### Neither affected:* and caching:* help with the worst case scenario

Affected rebuilds only what can be broken (on average 20-30% of projects in the repo). Caching helps not to rerun anything twice. However, the worst case scenario is when everything is affected and the cache is invalidated. Nx is smart, but once in a while you need to rebuild and retest everything.

The only way around is to distribute the CI. Nx comes with commands to make it easy. We have a few example repositories showing how to do it.

* [Example of setting up distributed Jenkins build for Nx workspace](https://github.com/nrwl/nx-jenkins-build)
* [Example of setting up distributed Azure build for Nx workspace](https://github.com/nrwl/nx-azure-build)
