# Understand the folder structure

Recall from [the previous chapter](./01-create-a-project.md) that `@codemod-utils/cli` creates a Node project. 

While Node gives you lots of freedom in organizing files, `@codemod-utils` asks you to follow several conventions. This will help with debugging issues and migrating your project, should we discover better solutions in the future.

Goals:

- Familiarize with the folder structure
- Familiarize with conventions from `@codemod-utils`


## Folder structure

Let's take a look how `ember-codemod-rename-test-modules` is structured as a tree. For simplicity, the tree only shows what's important for the tutorial.

```sh
ember-codemod-rename-test-modules
├── bin
│   └── ember-codemod-rename-test-modules.ts
├── src
│   ├── (blueprints)
│   ├── steps
│   ├── types
│   ├── (utils)
│   └── index.ts
├── tests
│   ├── fixtures
│   ├── helpers
│   ├── index
│   ├── steps
│   └── (utils)
└── codemod-test-fixtures.sh

# Hidden: dist, dist-for-testing, tmp
```


### bin

The `bin` folder has an **executable file**. This file allows your users ("end-developers") to run the codemod either with `npx`, or locally after they modify the clone of your repo.

```sh
# Compile TypeScript
pnpm build

# Run codemod
./dist/bin/ember-codemod-rename-test-modules.js --root <path/to/your/project>
```

It also means, you can test the codemod on a project on your local machine.

```sh
./dist/bin/ember-codemod-rename-test-modules.js --root ../../work-projects/client
```


### src

The `src` folder includes your **source code**.

The following list, which explains the `src` folder in detail, has many items. But no worries, they will make more sense once you complete the tutorial.

- The main entry point is `src/index.ts`. This file is a good place to start when studying another person's codemod.

    <details>

    <summary>Example: <code>src/index.ts</code></summary>

    ```ts
    export function runCodemod(codemodOptions: CodemodOptions): void {
      const options = createOptions(codemodOptions);

      // TODO: Replace with actual steps
      addEndOfLine(options);
    }
    ```

    </details>


- A codemod must break a large problem into small **steps**. Each step corresponds to exactly 1 file in the `src/steps` folder.

    In addition, each file (1) name-exports (2) a function that (3) has a descriptive name and (4) runs synchronously. In other words, we avoid premature abstractions and optimizations in favor of simplicity and reliability.

    <details>

    <summary>Example: <code>src/steps/add-end-of-line.ts</code></summary>

    ```ts
    export function addEndOfLine(options: Options): void {
      // ...
    }
    ```

    </details>

    The steps are re-exported in `src/steps/index.ts` so that `src/index.ts` (and tests) can easily consume them.

    <details>

    <summary>Example: <code>src/steps/index.ts</code></summary>

    ```ts
    export * from './add-end-of-line.js';
    export * from './create-options.js';
    ```

    </details>

    To recap, each step is a function.

- A step is encouraged to have smaller sub-steps (sub-steps are also functions), if it improves the project's maintability (fewer lines of code per file) and extensibility (group logically related steps). However, try to avoid premature abstractions.

    A sub-step lives in `src/steps/<step-name>/<sub-step-name>.ts`. It may be re-exported in `src/steps/<step-name>/index.ts`.

    You will find an example of breaking a step into smaller steps towards the end of this tutorial.

- This tutorial doesn't cover **blueprints**, files that you can use like a "stamp" to create or update certain files in a project. Blueprints must live in the `src/blueprints` folder. The CLI will create this folder (along with a few other files) for you.

- You may extract "utility" functions from a step(s) so that you can write unit tests. However, try to avoid premature abstractions.

    The functions must live in the `src/utils` folder. Similarly to in an Ember project, you have some freedom in how you organize files inside this folder.

    You will find an example of **utilities** towards the end of this tutorial.


### tests

The `tests` folder includes your **tests**, **fixtures** (files that represent your end-consumer's project), and **test helpers** (things that help you write tests).

Again, there are some conventions:

- Test files must have the file extension `*.test.ts`.

    Each file should have only 1 test, and each test only 1 assertion (with the exception of two assertions to check idempotency). The goal is to write tests that are simple.

    <details>

    <summary>Example: <code>tests/index/sample-project.test.ts</code></summary>

    ```ts
    import { runCodemod } from '../../src/index.js';

    test('index > sample-project', function () {
      loadFixture(inputProject, codemodOptions);

      runCodemod(codemodOptions);

      assertFixture(outputProject, codemodOptions);

      // Check idempotence
      runCodemod(codemodOptions);

      assertFixture(outputProject, codemodOptions);
    });
    ```

    </details>

    <details>

    <summary>Example: <code>tests/steps/add-end-of-line/base-case.test.ts</code></summary>

    ```ts
    import { addEndOfLine } from '../../../src/steps/index.js';

    test('steps | add-end-of-line > base case', function () {
      loadFixture(inputProject, codemodOptions);

      addEndOfLine(options);

      assertFixture(outputProject, codemodOptions);
    });
    ```

    </details>

    You have some freedom in how you name tests. For example, you might use the `|` and `>` characters to follow the conventions from Ember and QUnit.

- Similarly to in an Ember project, we write tests at the "acceptance," "integration," and "unit" levels. Broadly speaking, the tests in `tests/index`, `tests/steps`, and `tests/utils` correspond to these levels, respectively.

    For `tests/steps`, the folder structure should match that of `src/steps`. The same goes for `tests/utils`.

- Writing tests for sub-steps is _discouraged_. Instead, write tests for the parent step (integration) or for the related utilities (unit). By doing so, we can easily change sub-steps (often, an implementation detail) in the future.

- Fixture files must live in the `tests/fixtures` folder.

    A fixture project for an acceptance test lives in the folder `tests/fixtures/<fixture-name>`, while that for an integration test in `tests/fixtures/steps/<test-case-name>`.

    <details>

    <summary>Example: <code>tests/fixtures/sample-project/index.ts</code></summary>

    ```ts
    import { convertFixtureToJson } from '@codemod-utils/tests';

    const inputProject = convertFixtureToJson('sample-project/input');
    const outputProject = convertFixtureToJson('sample-project/output');

    export { inputProject, outputProject };
    ```

    </details>

- Test helpers must live in the `tests/helpers` folder.

    <details>

    <summary>Example: <code>tests/helpers/shared-test-setups/sample-project.ts</code></summary>

    `codemodOptions` and `options` are inputs that almost every step needs. We can define these for a given fixture project (e.g. one that is named `sample-project`).

    ```ts
    import type { CodemodOptions, Options } from '../../../src/types/index.js';

    const codemodOptions: CodemodOptions = {
      projectRoot: 'tmp/sample-project',
    };

    const options: Options = {
      projectRoot: 'tmp/sample-project',
    };

    export { codemodOptions, options };
    ```

    </details>

Note, the conventions for folder names may change in the future, so that we can simplify onboarding for people who have a background in Ember. The ideas (e.g. writing tests at different levels, using fixture files to test the codemod thoroughly) will remain the same.


### dist, dist-for-testing, tmp

To run the codemod (written in TypeScript), it must be compiled to JavaScript first.

Running the `build` script (re)creates the `dist` folder. The files in this folder are what is shipped to the end-developers.

Running the `test` script (re)creates the `dist-for-testing` folder. The files in this folder are what is tested. The fixture files are created in the `tmp` folder.


### codemod-test-fixtures.sh

The acceptance tests will likely fail after you create or update a step. By running this shell script, you can automatically update fixture files and get the acceptance tests to pass.

```sh
./codemod-test-fixtures.sh
```

The recommended workflow is:

1. Create or update a step.
1. Commit the code change.
1. Run the shell script.
1. Commit the code change.

This way, the second commit clearly shows the step's effect.

Note, the fixture files for that step and the subsequent steps (i.e. fixture files at the integration level) will need to be updated manually.


<div align="center">
  <div>
    Next: <a href="./03-sketch-out-the-solution.md">Sketch out the solution</a>
  </div>
  <div>
    Previous: <a href="./01-create-a-project.md">Create a project</a>
  </div>
</div>