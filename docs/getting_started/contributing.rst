Getting Started & Contributing
==============================

Welcome to the QSET Documentation Hub! This guide will help you set up the documentation locally, learn how to contribute, and understand the "Docs-as-Code" workflow.

The "First 24 Hours" Guide
--------------------------

If you are a new member, follow these steps to get your environment ready:

1. **Install Prerequisites**:
   * **Python 3.10+**: Ensure Python is installed and added to your system's PATH.
   * **Git**: Install Git to clone the repository and manage version control.

2. **Clone the Repository**:
   Open your terminal (Command Prompt or PowerShell on Windows) and run:

   .. code-block:: bash

      git clone https://github.com/QSET-Sat-Launch-1/qset-doc.git
      cd qset-doc

3. **Set Up a Virtual Environment (Recommended)**:
   Creating a virtual environment keeps your project dependencies isolated.

   .. tab-set::

      .. tab-item:: Windows

         .. code-block:: powershell

            python -m venv .venv
            .venv\Scripts\activate

      .. tab-item:: macOS / Linux

         .. code-block:: bash

            python3 -m venv .venv
            source .venv/bin/activate

4. **Install Dependencies**:
   With your virtual environment active, install the required packages:

   .. code-block:: bash

      pip install -r requirements.txt

5. **Build and Preview the Documentation**:
   Build the files to HTML and preview them in your browser.

   .. tab-set::

      .. tab-item:: Windows

         .. code-block:: powershell

            .\make.bat serve

         *(This builds the site and automatically opens it in your default browser. To view future changes, run this command again.)*

      .. tab-item:: macOS / Linux

         .. code-block:: bash

            sphinx-build -b html docs docs/_build/html
            open docs/_build/html/index.html

         *(If you are on Linux and ``open`` is not available, use ``xdg-open docs/_build/html/index.html``.)*


Using the Documentation Tools
-----------------------------

We use several Sphinx extensions to make our documentation interactive and professional.

Mermaid Diagrams
^^^^^^^^^^^^^^^^
Use Mermaid to draw flowcharts, state machines, and architectures directly in text.

.. code-block:: rst

   .. mermaid::

      graph TD
         A[Start] --> B{Is it working?};
         B -- Yes --> C[Great!];
         B -- No --> D[Debug];

Sphinx Design (UI Elements)
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Use grids, cards, and tabs to organize content cleanly.

.. code-block:: rst

   .. tab-set::

      .. tab-item:: Windows

         Windows instructions here.

      .. tab-item:: Linux

         Linux instructions here.

Heritage & Continuity
---------------------

As a student team, it's critical to preserve knowledge when members graduate. Please follow these guidelines:

1. **Lessons Learned Logs**: At the end of every project or testing phase, add a "Lessons Learned" section to your subteam's documentation. Document what failed, why it failed, and how you fixed it.
2. **Component Status Badges**: Clearly label hardware designs. Are they `Flight Proven`, `Prototype`, or `Experimental`?
3. **Traceability (The V-Model)**: Ensure every piece of code or hardware traces back to a system requirement. See the :doc:`/standards/v_model_example` for more details.

Adding New Subteam Documentation
--------------------------------

We want adding documentation to be as frictionless as possible. You do not need to edit any configuration or index files to make your new pages appear.

To add documentation for a new project/subsystem:

1. **Locate your subteam's folder** under ``docs/subteams/[subteam_name]/`` (e.g., [docs/subteams/adcs/](file:///c:/Users/Jaron/Downloads/QSET_Doc/docs/subteams/adcs/)).
2. **Create a new file** ending in ``.rst`` (or ``.md``). For example: ``sensor_calibration.rst``.
   * *Tip*: You can copy the design, status badges, Mermaid, and Sphinx-Needs templates from the [Project Template](file:///c:/Users/Jaron/Downloads/QSET_Doc/docs/subteams/project_template.rst) to get started quickly.
3. **Start writing!** The subteam's main index page uses a wildcard glob (``*``) in its table of contents. Your new file will be detected and added to the sidebar automatically the next time you build the site.

Docs-as-Code Workflow
---------------------

To make changes and push them to the team repository, follow this step-by-step workflow:

1. **Pull the Latest Changes**:
   Always make sure you start with the newest version of the codebase.

   .. code-block:: bash

      git checkout main
      git pull origin main

2. **Create a New Branch**:
   Never make edits directly on ``main``. Create a descriptive branch for your work (e.g., ``feat-obc-telemetry`` or ``docs-adcs-sensors``).

   .. code-block:: bash

      git checkout -b your-branch-name

3. **Edit and Preview**:
   * Open the workspace folder in your text editor (like VS Code).
   * Edit existing files or create new ones under the appropriate subteam folder.
   * Re-build and preview your changes locally to verify there are no formatting or build errors:

     * **Windows**: ``.\make.bat serve``
     * **macOS/Linux**: ``sphinx-build -b html docs docs/_build/html && open docs/_build/html/index.html``


4. **Commit Your Changes**:
   Stage and commit your edits.

   .. code-block:: bash

      git add .
      git commit -m "Briefly describe your documentation changes"

5. **Push and Open a Pull Request**:
   Push your new branch to GitHub and request feedback.

   .. code-block:: bash

      git push origin your-branch-name

   Go to the `qset-doc GitHub repository <https://github.com/QSET-Sat-Launch-1/qset-doc>`_ in your browser. You will see a prompt to **Compare & pull request**. Click it, write a short description of what you added/modified, and submit it!
   
   Once approved and merged by a subteam lead, our GitHub Actions pipeline will automatically build and deploy your updates to the live site.

