# Storybook Codemods

Storybook Codemods is a collection of codemod scripts written with JSCodeshift.
It will help you migrate breaking changes & deprecations.

## CLI Integration

The preferred way to run these codemods is via the CLI's `migrate` command.

To get a list of available codemods:

```
npx -p @storybook/cli sb migrate --list
```

To run a codemod:

```
npx -p @storybook/cli sb migrate name-of-codemod --glob "**/*.stories.js"
```

## Installation

If you want to run these codemods by hand:

```sh
yarn add jscodeshift @storybook/codemod --dev
```

- `@storybook/codemod` is our collection of codemod scripts.
- `jscodeshift` is a tool we use to apply our codemods.

After running the migration commands, you can remove them from your `package.json`, if you added them.

## How to run a codemod script

From the directory where you installed both `jscodeshift` and `@storybook/codemod` run:

Example:

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/update-organisation-name.js . --ignore-pattern "node_modules|dist"
```

Explanation:

    <jscodeShiftCommand> -t <transformFileLocation> <pathToSource> --ignore-pattern "<globPatternToIgnore>"

## Transforms

### update-organisation-name

Updates package names in imports to migrate to the new package names of storybook.

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/update-organisation-name.js . --ignore-pattern "node_modules|dist"
```

There's a mapping of paths we replace but this example explains the gist of it:

Example:

```js
import { storiesOf } from '@kadira/storybook';
import { linkTo } from '@kadira/storybook-addon-links';
```

Becomes

```js
import { storiesOf } from '@storybook/react';
import { linkTo } from '@storybook/addon-links';
```

### update-addon-info

Replaces the Info addon's deprecated `addWithInfo` API with the standard `withInfo` API.

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/update-addon-info.js . --ignore-pattern "node_modules|dist"
```

Simple example:

```js
storiesOf('Button').addWithInfo('simple usage', 'This is the basic usage of the button.', () => (
  <Button label="The Button" />
));
```

Becomes

```js
storiesOf('Button').add(
  'simple usage',
  withInfo('This is the basic usage of the button.')(() => <Button label="The Button" />)
);
```

With options example:

```js
storiesOf('Button').addWithInfo(
  'simple usage (disable source)',
  'This is the basic usage of the button.',
  () => <Button label="The Button" />,
  { source: false, inline: true }
);
```

Becomes

```js
storiesOf('Button').add(
  'simple usage (disable source)',
  withInfo({
    text: 'This is the basic usage of the button.',
    source: false,
    inline: true,
  })(() => <Button label="The Button" />)
);
```

### add-component-parameters

This tries to smartly adds "component" parameters to all your existing stories
for use in SB Docs.

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/add-component-parameters.js . --ignore-pattern "node_modules|dist"
```

For example:

```js
input { Button } from './Button';
storiesOf('Button', module).add('story', () => <Button label="The Button" />);
```

Becomes:

```js
input { Button } from './Button';
storiesOf('Button', module)
  .addParameters({ component: Button })
  .add('story', () => <Button label="The Button" />);
```

Heuristics:

- The storiesOf "kind" name must be Button
- Button must be imported in the file

### convert-storiesof-to-module

This converts all of your "old-style" `storiesOf` stories into component module format, which uses standard ES6 modules.

> NOTE: The output of this transformation may require manual editing after running the transformation. `storiesOf` format allows multiple "kinds" (components) to be declared per file, but module format only allows a single component per file. Therefore, if you use this feature in your input stories, you will need to split up the resulting outputs by hand. You'll see a warning at the console if you need to hand edit.

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/convert-storiesof-to-module.js . --ignore-pattern "node_modules|dist"
```

For example:

```js
storiesOf('Button', module)
  .add('story', () => <Button label="Story 1" />)
  .add('second story', () => <Button label="Story 2" onClick={action('click')} />)
  .add('complex story', () => (
    <div>
      <Button label="The Button" onClick={action('onClick')} />
      <br />
    </div>
  ));
```

Becomes:

```js
export default {
  title: 'Button',
};

export const story = () => <Button label="Story 1" />;

export const story2 = () => <Button label="Story 2" onClick={action('click')} />;
story2.story = { name: 'second story' };

export const story3 = () => (
  <div>
    <Button label="The Button" onClick={action('onClick')} />
    <br />
  </div>
);
story3.story = { name: 'complex story' };
```

Heuristics:

- If a file has any default export, it will be skipped
- If a file has multiple `storiesOf` declarations, it will convert each one seperately. This generates invalid ES6, but you can edit the file by hand to split it into multiple files (or whatever is appropriate).

### convert-module-to-mdx

This converts all of your component module stories into MDX format, which integrates story examples and long-form documentation.

> NOTE: The output of this transformation may require manual editing after running the transformation. MDX is finnicky about the top-level statements it allows. For example, [variables should be defined with exports](https://mdxjs.com/getting-started/#defining-variables-with-exports), meaning `const foo = 5;` should be rewritten as `export const foo = 5;`. We don't do this transformation automatically, since you may prefer to refactor your stories.

```sh
./node_modules/.bin/jscodeshift -t ./node_modules/@storybook/codemod/dist/transforms/convert-to-module-format.js . --ignore-pattern "node_modules|dist"
```

```js
export default {
  title: 'Button',
};

export const story = () => <Button label="Story 1" />;

export const story2 = () => <Button label="Story 2" onClick={action('click')} />;
story2.story = { name: 'second story' };
```

Becomes:

```md
import { Meta, Story } from '@storybook/addon-docs/blocks';

# Button

<Meta title='Button'>

<Story name='story'><Button label="Story 1" /></Story>

<Story name='second story'>
  <Button label="Story 2" onClick={action('click')} />
</Story>
```
