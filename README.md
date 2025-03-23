# What About Adarsh

This is the source code for the "What About Adarsh" website, built using [Hugo](https://gohugo.io/), a fast and flexible static site generator. The site uses the Ananke theme and is automatically built and deployed using GitHub Actions.

## Project Structure

- `content/`: Contains the content files for the site.
- `data/`: Contains data files used by the site.
- `layouts/`: Contains custom layouts for the site.
- `static/`: Contains static files such as images, CSS, and JavaScript.
- `themes/`: Contains the Ananke theme used by the site.
- `config.toml`: The main configuration file for the site.
- `.github/workflows/hugo.yml`: The GitHub Actions workflow file for building and deploying the site.

## Prerequisites

- [Hugo](https://gohugo.io/getting-started/installing/) (latest version)
- [Git](https://git-scm.com/)

## Getting Started

1. Clone the repository:

    ```sh
    git clone https://github.com/thatsmeadarsh/whataboutadarsh.git
    cd whataboutadarsh
    ```

2. Install Hugo:

    Follow the [installation guide](https://gohugo.io/getting-started/installing/) to install Hugo on your machine.

3. Run the Hugo server:

    ```sh
    hugo server
    ```

4. Open your browser and visit `http://localhost:1313` to see the site.

## Deployment

The site is automatically built and deployed using GitHub Actions. The workflow is defined in `.github/workflows/hugo.yml`. The workflow is triggered by various events such as pushes to the main branch, pull requests to the main branch, manual triggers via the GitHub UI, and repository dispatch events.

### Workflow Steps

1. **Checkout code**: Checks out the repository's code, including submodules.
2. **Setup Hugo**: Sets up Hugo using the specified version.
3. **Download Remote JSON**: Downloads a remote JSON file and saves it to the `data/` directory.
4. **Commit data folder**: Commits any changes in the `data/` folder to the repository.
5. **Push changes**: Pushes the changes to the main branch.
6. **Delete Hugo cache**: Deletes the Hugo cache to ensure a clean build.
7. **Build Hugo site**: Builds the Hugo site.
8. **Commit public folder**: Commits any changes in the `public/` folder to the repository.
9. **Push changes**: Pushes the changes to the main branch.
10. **Clone target repository**: Clones the target repository and clears its contents.
11. **Commit and push to target repository**: Copies the newly generated site files into the target repository and pushes the changes.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request for any changes or improvements.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## This is AI Generated and not yet reviewed ##
