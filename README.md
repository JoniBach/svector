# Svector

[Svector on npm](https://www.npmjs.com/package/svector)

**Svector** is an **SVG editor** component for [Svelte](https://svelte.dev/), built on top of the powerful [Paper.js](http://paperjs.org/) library. It provides a user-friendly interface with tools for drawing paths, creating shapes, adding text, and exporting your work—all via a sleek, customizable toolbar.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
    - [Basic Example](#basic-example)
    - [Extended Demo](#extended-demo)
    - [Custom Actions](#custom-actions)
- [API](#api)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- **Paper.js Integration**: Leverages a robust vector graphics framework for high-performance drawing and editing.
- **Essential Tools**: Built-in pen (vector) tool, shape tools, text insertion, and upload/download features.
- **Event Handling**: Listens to `save` events and other callbacks for easy integration into your workflow.
- **Customizable Toolbar**: Easily add or remove tools using an `actions` array, or create your own custom buttons.
- **Minimal Setup**: Only a few lines of Svelte code needed to embed the component in your app.

---

## Installation

Install **Svector** with your favorite package manager:

```bash
npm install svector
```

> **Note**: Since Svector is built on top of [Paper.js](http://paperjs.org/), it will automatically install any necessary dependencies.

---

## Usage

### Basic Example

Below is the minimal setup to integrate **Svector** into your Svelte application. This example includes a download button and the vector (pen) tool.

```svelte
<script>
    import { Svector } from 'svector';

    // Handle the save event
    function handleSave() {
        alert('File saved!');
    }
</script>

<Svector
    on:save={handleSave}
    file=""
    actions={[
        { type: 'download' },
        { type: 'vector' }
    ]}
/>
```

- **`file`**: The SVG content you want to load initially (string).
- **`actions`**: An array defining the toolbar buttons. Available `type`s include `download`, `upload`, `text`, `shape`, `vector`, and custom actions.

---

### Extended Demo

Here’s a more feature-rich setup demonstrating multiple default tools plus a custom action:

```svelte
<script>
    import { Svector } from 'svector';

    let file = ''; // Could be an SVG string you’ve saved or fetched from somewhere

    // Define a list of actions, including default and custom
    const actions = [
        { type: 'upload' },
        { type: 'download' },
        { type: 'text' },
        { type: 'shape' },
        { type: 'vector' },
        {
            type: 'custom',
            name: 'Generate Image',
            icon: 'mdi:sparkles',
            action: () => generateImage()
        }
    ];

    async function handleSave(event) {
        console.log('Save event triggered:', event.detail);
        // Do something with the event data, e.g. store the file
    }

    async function generateImage() {
        const promptText = prompt('Enter a prompt for image generation:');
        if (promptText !== null) {
            alert(`User prompt: ${promptText}`);
            // Implement your custom logic here
        }
    }
</script>

<Svector
    width="500px"
    height="500px"
    on:save={handleSave}
    {file}
    {actions}
/>
```

This setup shows a toolbar with:

- **Upload**: Allows you to upload an existing SVG or image.
- **Download**: Exports the current drawing as an SVG file.
- **Text**: Adds a text field to your canvas.
- **Shape**: Inserts shapes (e.g., rectangles, circles) on the canvas.
- **Vector**: Activates the pen (vector) drawing tool.
- **Generate Image**: A custom action triggering your custom logic.

---

### Custom Actions

Svector supports custom actions via the `type: 'custom'` option in the `actions` array. Each custom action must have:

- **`type: 'custom'`**: Indicates it’s a custom button.
- **`name`**: The button label displayed in the toolbar.
- **`icon`**: A string referencing an [Iconify](https://icon-sets.iconify.design/) icon.
- **`action`**: A callback function that executes when the button is clicked.

```js
{
    type: 'custom',
    name: 'Generate Image',
    icon: 'mdi:sparkles',
    action: () => console.log('Custom button clicked!')
}
```

Use these to integrate your own functionality—like AI image generation, custom exports, or other advanced workflows.

---

## API

Svector provides a single Svelte component: `Svector`.

### Props

- **`file`** (string): Initial SVG file content to load into the editor.
- **`actions`** (array): Array of tool/action objects, each describing a toolbar button. Recognized `type` values include:
    - `upload`
    - `download`
    - `text`
    - `shape`
    - `vector`
    - `custom`
- **`width`** (string): Width of the editor container, e.g., `"800px"` or `"100%"`.
- **`height`** (string): Height of the editor container, e.g., `"600px"` or `"100vh"`.

### Events

- **`save`**: Dispatched when the user triggers a save operation (e.g., via a custom or built-in action). Provides an event detail containing the current SVG state.

```svelte
<Svector
    on:save={(event) => {
        const svgContent = event.detail;
        console.log('SVG content:', svgContent);
    }}
/>
```

---

## Roadmap

We plan to expand Svector’s capabilities by exposing even more internal operations from Paper.js, as well as providing more customization options such as:

- **Advanced Pen/Path Tools** (Bezier curves, shape recognition)
- **Multi-Layer Support**
- **Customizable Keyboard Shortcuts**
- **Plugin System** for third-party additions

---

## Contributing

Contributions are always welcome!

1. [Fork the repository](https://github.com/JoniBach/svector)
2. Create your feature branch: `git checkout -b feature/my-new-feature`
3. Commit your changes: `git commit -m 'Add some feature'`
4. Push to the branch: `git push origin feature/my-new-feature`
5. Open a Pull Request on GitHub

---

## License

This project is licensed under the [MIT License](https://github.com/JoniBach/svector/blob/main/LICENSE). You are free to use and modify it for personal or commercial projects.

---

**Happy Vector Editing!** If you have any questions or suggestions, feel free to create an issue or pull request on [GitHub](https://github.com/JoniBach/svector).
