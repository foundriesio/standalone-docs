# Standalone-docs

This repository houses documentation not associated with a specific code repository.
This includes stand-alone pages and guides that involve *multiple* repos.

While documents within this repo can be linked to for direct viewing,
it is structured around the idea that documents will be cloned/copied as needed;
processing can then be done by a Static Site Generator or PDF builder.

## Layout and Structure

Text files go into a folder with the document name/top header.
This is where following the guidance to use unique headers/document titles is required,
avoiding clashes.

A document can use subdirectories and can have assets such as images as required.

## Workflow

Documents should be linted locally,
using a vale configuration matching the requirements for your document.
The linter action for this repo will only enforce rules that will apply to all documents.

To add a document, fork, work on a branch, and then open a PR against `main`.

Please add the required frontmatter meta-data to all files.

## Guides

- [Set Up UNO Q with Qualcomm Linux and a Local Update Server](uno-q-setup-and-updates/README.md): build, flash, register, and update an Arduino UNO Q.
