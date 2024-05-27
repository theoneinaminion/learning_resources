## Linux Compilers
1. [MikTex](https://miktex.org/): Lightweight, automatically installs what you need from internet. 
2. [TexLive](https://tug.org/texlive/): More generally used. 

    ```
       sudo apt-get update
       sudo apt install perl
       sudo apt-get install texlive
    ```   

Above is the lightweight installation. The full version can be installed as follows:
    `sudo apt-get install texlive-full`
**WARNING**: Full version is installs all the available language that makes the installation very heavy (~7GB).  


## Popular Editors
1. [Overleaf](https://www.overleaf.com/) 
    - Online. Use it from your browser.
    - No latex installation required from users end. 
    - Thousands of templates available to be used readily. 
    - Easy collaboration/Feedback system
    - Free version is getting increasingly restrictive.
    Very good for sharing your document, giving feedbacks. However, it is getting increasingly restrictive for free users. 

2. [TexStudio](https://www.texstudio.org/#home) - 
    - Need to install latex compilers manually.
    - Offline, open-source, and will find most of overleaf's features. 
  
3. [VS Code]: VS Code could be used to write, edit, compile documents in LaTex. The following can be followed to set it up in WSL/Linux environments: 

   - TexLive is recommended. Skip the below 3 steps if you have TexLive is installed and start from installing `latexmk`. 
    ```
        sudo apt-get update
        sudo apt install perl
        sudo apt-get install texlive

    ```  

    `sudo tlmgr install latexmk`
    
   - Add the TexLive to `PATH` variable in `.bashrc` by doing something this (this is very important):

    `export PATH=$PATH:/usr/local/texlive/YYYY/bin/PLATFORM`

    For a latest 2024 version on a `x86_64-linux` platform, it will look something like:

   `export PATH=$PATH:/usr/local/texlive/2024/bin/x86_64-linux`

   (For VS Code Remote Development, you might want to set the `PATH` in `.bash_profile` or `.profile`)

   - Install the [LaTex Workshop](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop) extension in VS Code. 
  
   - It is highly recommended to consult the manual of [LaTex Workshop](https://github.com/James-Yu/LaTeX-Workshop/wiki) before starting. It has amazing shortcuts which can make it easier to write documents.
  
   - **Disadvantage**: Does not have templates like in Overleaf, TexStudio.
  
   - **Potential Solution**:
     - Using [Snippets](https://code.visualstudio.com/docs/editor/userdefinedsnippets) in VS Code. 
     - Snippets can be used to insert repeatitive code blocks in VS Code. 
     - Easily generate your snippets using an [Online Generator](https://snippet-generator.app/?description=&tabtrigger=&snippet=&mode=vscode). 


## Other Handy LaTex Tools: 

1. [Latex Equation Editor](https://www.latex4technics.com/): Write equations and generate latex code automatically. 
2. [Latex Table Generator](https://www.tablesgenerator.com/): Generate and edit latex tables easily. 


 



