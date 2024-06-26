# Futuring Machines

A tool for Human-AI collaborative writing of speculative fiction stories.

## Quick Start

- [download Ollama](https://ollama.com/download)
- install mistral: `ollama pull mistral`
- clone this repository
- install dependencies: `npm install`
- start development server: `npm run dev`

## Project Setup

### Requirements

- [Node.js](https://nodejs.org/en)
- [Ollama](https://ollama.com) or an API exposing a comparable `api/generate` [endpoint](https://github.com/ollama/ollama/blob/main/docs/api.md#generate-a-completion).

### Interface

install dependencies

```sh
npm install
```

compile and hot-reload for development

```sh
npm run dev
```

compile and minify for production

```sh
npm run build
```

lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```

### Specify Model and API URL

Open the `.env` file and set the `VITE_MODEL` and `VITE_API_URL` environment variables accordingly. When using Ollama you'll need to pull the model before using it.

```
VITE_MODEL=mistral
VITE_API_URL=http://localhost:11434/api/generate
```

### Adding Story Templates

Story templates serve as a starting point for writers. They can be static text, text generated by a LLM, or a mix of both.

Story templates are json files located in `src/assets/templates` and imported in `src/assets/templates/index.js`.

Each story template requires a `name` and one or more `actions`. Each action requires a `type` (`static` or `generate`) and a `template`. Depending on the type the template will either just return the template text or send it to the LLM to generate the text response.

A simple template for static text:

```json
{
  "name": "static design template",
  "actions": [
    {
      "type": "static",
      "template": "In a world where technology has advanced beyond our wildest dreams, design practices have evolved to keep pace. As we step into the future, designers are no longer confined by the limitations of physical materials and two-dimensional screens."
    }
  ]
}
```

A simple template for text generation

```json
{
  "name": "generated design template",
  "actions": [
    {
      "type": "generate",
      "template": "Write the first paragraph of a story on the state of design practice in 2100."
    }
  ]
}
```

For easy customization you can specify variable in `env` and envoke them in the template string using the syntax `::variable::`.

```json
{
  "name": "generated design template",
  "actions": [
    {
      "type": "generate",
      "template": "Write the first paragraph of a story on the state of ::topic:: practice in ::year::."
    }
  ],
  "env": {
    "year": 2100,
    "topic": "design"
  }
}
```

It is also possible to chain multiple actions. The example below inserts generated text into a static template. The first action uses the `bind` property which binds the generated response to the specified variable `reply`. The static action inserts that variable by using `::reply::`

```json
{
  "name": "mixed design template",
  "actions": [
    {
      "type": "generate",
      "template": "Kim is a designer living in the year 2100. A friend asks her what she does all day. What does she respond?",
      "bind": "reply"
    },
    {
      "type": "static",
      "template": "Kim explains \"::reply:: Is this something you'd like to do as well, Alex?\""
    }
  ]
}
```

### Adding Prompts

Prompts allow writers to interact with the LLM. They are similarily structured to story templates and can currently either be executed on selected text or new lines.

Like story templates they require a name and a list of actions. Additionally a trigger `selection` or `new-line` and a mode `replace` or `append` must be specified.

##### Example: Adding a mode that is triggered on new line

Here's a simple example to continue the story. The response will be appended at the end of the existing text. Here you can use the flag `::full::`, which will be replaced by the full story text.

````json
{
  "name": "continue writing",
  "trigger": "new-line",
  "mode": "append",
  "actions": [
    {
      "type": "generate",
      "template": "Continue the following story, which is delimited with triple backticks. \n\nStory: ```::full::```"
    }
  ]
}
````

**Note:** Notice in the example above the use of triple backticks around `::full::`. When prompting it is important to write clear and specific instructions, especially when prompts get long. One tactic is using delimiters to clearly indicate distinct parts of the input. Delimiters can be anything like triple backticks (```), triple dashes (---), angle brackets (< >) or XML tags (<tag> </tag>), among others.

##### Example: Adding a mode that is triggered on text selection and that will replace the selected text

Here's a simple example to shorten a selected text. The response will replace the selected text. `::selection::` in the template string will be replaced with the user selected text. `::selection::` is only available if the trigger is set to `selection`. Here you can also use `::full::` which is replaced by the full story text. `::full::` is available on all triggers.

```json
{
  "name": "condense",
  "trigger": "selection",
  "mode": "replace",
  "actions": [
    {
      "type": "generate",
      "template": "Shorten the following text: ::selection::"
    }
  ]
}
```

##### Example: Adding a mode that is triggered on text selection and that will append text at the end of the story

Here's a simple example to make the model further elaborate the idea (term, concept, etc.) behind a selected text. The response will be appended at the end of the story. `::selection::` in the template string will be replaced with the user selected text. Notice that you can also use `::full::`, which is replaced by the full story text. `::full::` is available on all triggers.

```json
{
  "name": "elaborate",
  "trigger": "selection",
  "mode": "append",
  "actions": [
    {
      "type": "generate",
      "template": "Continue the following story further elaborating the aspect addressed in the following text. Text: '::selection::'. \n\nStory: ::full::"
    }
  ]
}
```

##### Example: Chaining multiple actions in a same interaction mode

Again, it is possible to chain multiple actions. The action type `options` allows users to select from given options. The `bind` property is used to define a variable `option` which the selected option will be assigned to. In the second action the selected option is inserted into the template string using `::option::`

```json
{
  "name": "push forward",
  "trigger": "selection",
  "mode": "replace",
  "actions": [
    {
      "type": "options",
      "options": ["10 years", "100 years", "1000 years", "10000 years"],
      "bind": "option"
    },
    {
      "type": "generate",
      "template": "rewrite the following text as if it was set ::option:: in the future ::selection::"
    }
  ]
}
```

##### Example: Tell the model to generate options

Alternatively, options can also be generated using the action type `generate-options`. The example below first offers static options, followed by generated options and finally uses both selections to generate text.

When using `generate options` it is crucial to state in the template string that the answer should be returned in json along with a set of keys. Additionally the keys must be listed under the `keys` property. These values are used to parse and verify the LLM response. Once the user chooses an option the keys and their respective value become available as variables. The `name` property identifies the key which is used for presenting the options to the user.

```json
{
  "name": "diverge",
  "trigger": "new-line",
  "mode": "append",
  "actions": [
    {
      "type": "options",
      "options": ["poltical", "societal", "cultural"],
      "bind": "option"
    },
    {
      "type": "generate options",
      "template": "::full:: \n\n suggest three topic ideas to delvelop the above story further while focussing on ::option:: aspects. provide them in json with the following keys: topic, description",
      "keys": ["topic", "description"],
      "name": "topic"
    },
    {
      "type": "generate",
      "template": "::full:: \n\n continue the story above with one paragraph that focusses on ::option:: aspects and the topic of ::topic:: (::description::)"
    }
  ]
}
```

It might be more efficinet to generate options and final text response in one step. This can be achieved by rephrasing the `generate options` template string and closing with a `static` action that simply returns the value of one of the keys.

```json
{
  "name": "diverge alternative",
  "trigger": "new-line",
  "mode": "append",
  "actions": [
    {
      "type": "options",
      "options": ["positive", "neutral", "negative"],
      "bind": "tone"
    },
    {
      "type": "generate options",
      "template": "::full:: \n\n suggest three story coninuations to further tell the above story while keeping the tone ::tone::. limit the length of the continuations to 10 words each. provide them in json with the following keys: title, continuation",
      "keys": ["title", "continuation"],
      "name": "title"
    },
    {
      "type": "static",
      "template": "::continuation::"
    }
  ]
}
```

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur) + [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin).

## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).
