---
description: 'Markdown guidelines and content creation standards'
applyTo: '**/*.md'
---

# Markdown Content Rules

The following markdown content rules are enforced in the validators:

1. **Headings**: Use appropriate heading levels (H2, H3, etc.) to structure your content. Do not use an H1 heading, as this will be generated based on the title.
2. **Lists**: Use bullet points or numbered lists for lists. Ensure proper indentation and spacing.
3. **Code Blocks**: Use fenced code blocks for code snippets. Specify the language for syntax highlighting.
4. **Links**: Use proper markdown syntax for links. Ensure that links are valid and accessible.
5. **Images**: Use proper markdown syntax for images. Include alt text for accessibility.
6. **Tables**: Use markdown tables for tabular data. Ensure proper formatting and alignment.
7. **Line Length**: Limit line length to 400 characters for readability.
8. **Whitespace**: Use appropriate whitespace to separate sections and improve readability.
9. **Front Matter**: Include YAML front matter at the beginning of the file with required metadata fields.
10. **Diagrams**: Use Mermaid syntax for diagrams. Ensure that diagrams are properly formatted and rendered correctly.
11. **Breadcrumbs**: Include breadcrumbs at the top of the file to indicate the file's location within the project structure. Use proper markdown syntax for links in breadcrumbs.
12. **Indexing**: Ensure that the content is properly indexed and categorized for easy navigation. Always update the root README.md file to include links to new markdown files and sections as they are created.

## Formatting and Structure

Follow these guidelines for formatting and structuring your markdown content:

- **Headings**: Use `##` for H2 and `###` for H3. Ensure that headings are used in a hierarchical manner. Recommend restructuring if content includes H4, and more strongly recommend for H5.
- **Lists**: Use `-` for bullet points and `1.` for numbered lists. Indent nested lists with two spaces.
- **Code Blocks**: Use triple backticks (```) to create fenced code blocks. Specify the language after the opening backticks for syntax highlighting (e.g., `csharp`).
- **Links**: Use `[link text](URL)` for links. Ensure that the link text is descriptive and the URL is valid.
- **Images**: Use `![alt text](image URL)` for images. Include a brief description of the image in the alt text.
- **Tables**: Use `|` to create tables. Ensure that columns are properly aligned and headers are included.
- **Line Length**: Break lines at 80 characters to improve readability. Use soft line breaks for long paragraphs.
- **Whitespace**: Use blank lines to separate sections and improve readability. Avoid excessive whitespace.
- **Diagrams**: Use Mermaid syntax for diagrams. Ensure that diagrams are properly formatted and rendered correctly. Include a brief description of the diagram in the content.
- **Breadcrumbs**: Use `[Home](../README.md) > [Section](section.md) > [Subsection](subsection.md)` for breadcrumbs. Ensure that links in breadcrumbs are valid and point to the correct location within the project structure.
- **Indexing**: Ensure that the content is properly indexed and categorized for easy navigation. Always update the root README.md file to include links to new markdown files and sections as they are created.
