---
layout: posts
comments: true
author_profile: true
title: "A convoluted, but effective, python environments management setup for Emacs"
excerpt: Using various tools to make python environments working automagically also with emacs
---

**TL;DR**

*Using direnv, pyenv and a few other plugins you can automagically make your shell or Emacs activate and deactivate python environment as you step in or out project folders*

# Transitioning to Emacs made me reconsider my python environments setup
It's now 5+ months that I am using Emacs as my main IDE. Getting used to it has been an interesting experience, as well as a massive time sink. For now I don't see any reasons to go back to PyCharm, although I have never been a power user, so my perception of its value is probably a partial one. At the same time, given how steep the Emacs learning curve is, and how good modern IDEs are, I don't think I will sell Emacs as a worthy investment either.

Creating an effective, and modern IDEs with Emacs is a lot of work, which doesn't end with your Emacs config (you can find mine here [my github](https://github.com/matteo-pallini/emacs-config)). It also took me a while to make Emacs works effectively with different python projects and their related working environment. Specifically I wanted the LSP and various other code checkers to use the right python virtual environment when reading or editing python files belonging to different projects and environments. Eventually I succeded and this post is going to describe the final result.

## Ingredients
- [Direnv](https://direnv.net/) will automagically load environment variables or activate python virtual envs as soon as you get into directories containing an `.envrc` file. You can find [here](https://github.com/direnv/direnv/wiki/Python) the docs covering Python
- [Pyenv](https://github.com/pyenv/pyenv) will make very easy to install and manage various versions of python, and also specify which one to use in different environments.
- [Pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) will let you create virtualenv using the python versions available in pyenv.
- [Emacs direnv integration](https://github.com/wbolster/emacs-direnv) and [Pyvenv](https://github.com/jorgenschaefer/pyvenv) will be handy if you care about using the whole setup through Emacs as well
- Obviously [Poetry](https://python-poetry.org/), if that is your poison for managing and using virtual envs

## Final product
What do we gain by using so many tools?
- We can create python virtual envs using the specif python version we need
- We can create and manage them both using virtualenv (through pyenv) and poetry
- We can have them being activated/deactivated automactically as we enter/exit any directory of our choice
- Similarly Emacs will activate them automatically whenever we open a file in the directory of choice. Which means no more manual activation of them make the linting and LSP work

# Steps to follow
Install `direnv`, `pyenv`, `pyenv-virtualenv`, `emacs-direnv` and emacs `pyvenv` as per their documentation. Below I am also going to cover some specific configuration steps for the first two tools, the third shouldn't require any config, while for the fourth and fifth should be enough follow their docs.

### Pyenv configs
Now we need add to our preferred shell startup file (`.zshrc` and `zshenv` in my case) the environment variables needed for pyenv to work. Following the pyenv documentaion should be enough. There is also a useful snippet you can to your startup file so that the active environment name shows up in the terminal.

My `.zshrc` after this step
```
# Automatic venv activation
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"

# show on terminal the activate virtualenv name
setopt PROMPT_SUBST
show_virtual_env() {
  if [[ -n "$VIRTUAL_ENV" && -n "$DIRENV_DIR" ]]; then
    echo "($(basename $VIRTUAL_ENV))"
  fi
}
PS1='$(show_virtual_env)'$PS1

# needed for automatically activating virtual env
eval "$(pyenv virtualenv-init -)"
```

My `.zshenv` after this step
```
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
```

Also, when using Ubuntu this setup leads to the following error whenever I entered a directory containing a `.envrc` file
`direnv: PS1 cannot be exported. For more information see https://github.com/direnv/direnv/wiki/PS1`
Adding `unset PS1` at the end of the `.envrc` was enough to make it go away.

Worth mentioning that different shells may require slightly different scripts for showing the activate virtual env. This [resource](https://github.com/direnv/direnv/wiki/Python#restoring-the-ps1) could be helpful to figure out the right one for you.

### Direnv config
In order to make `direnv` work nicely with pyenv it will be necessary to create a `.direnvrc` file in your home and add the following script to it

```
use_python() {
    if [ -n "$(which pyenv)" ]; then
        local pyversion=$1
        pyenv local ${pyversion}
    fi
}

layout_virtualenv() {
    local pyversion=$1
    local pvenv=$2
    if [ -n "$(which pyenv virtualenv)" ]; then
        pyenv virtualenv --force --quiet ${pyversion} ${pvenv}-${pyversion}
    fi
    pyenv local --unset
}

layout_activate() {
    if [ -n "$(which pyenv)" ]; then
        source $(pyenv root)/versions/$1/bin/activate
    fi
}
```

and also add `eval "$(direnv hook zsh)"` to your shell startup script, again it will change depending on your preference.


### Other Misc
Just to give my whole setup. I also add two other environment variables, one to tell virtualwrapper where the environments are, and one to do the same for Poetry. Not that I have to necessarily use them, but it's nice to have them already there in case I want to.

## How to use this setup

At this point whenever you creat a new python project it is enough to do the followings.

Create a new environment `pyenv virtualenv 3.10.6 new-environment`

Create a `.envrc` file in the root directory of the new python project, and add the following two lines to it
```
layout activate new-environment
unset PS1
```
The last step is telling direnv that it can trust the newly create `.envrc` file by running `direnv allow`.

Finally you should be good to go and finally enjoy some magic!


## Use Poetry

Creating a poetry environment is then a simple as running `pyenv local your-python-version-of-interest` inside the poetry project folder and then creating the new project manually or through the standard poetry workflow.

# The final outcome
At this point you should have gain the following functionalities:
- As soon as you `cd` in the directory with the `.envrc` file the new environment will be active, you `cd` out and it's not active anymore
- Whenever you open a buffer in emacs for a python file living in that directory of the `.envrc` file the LSP and various linters you may use will now what python version and libraries to consider
- The above will work whether it's a python environment created with pyenv or with poetry

It took me some time to come up with this setup, but I found it to be a game changer when using emacs with python projects. I hope it may be so also for others, including you :-).

by the way these are the relevant bits in my emacs config
```
(use-package direnv
  :config
  (direnv-mode))

(use-package pyvenv
  :init
  (setenv "WORKON_HOME" (substitute-in-file-name "${WORKON_HOME}"))
  :config
  (pyvenv-mode 1))
```

## A small caveat
A slightly annoying thing of this approach is that it will create both python versions and virtual env in the same folder (ie `.pyenv/versions`). In case you don't want this to happen you can use the standard `venv` workflow directly by explicitly referring to the python version of interest. Something like this `~/.pyenv/versions/3.10.6/bin/python -m venv ~/.pyenv/versions/my-new-env` or alternatively `pyenv local 3.10.6 && python -m venv ~/.pyenv/versions/my-new-env`

Time taken to write post: 4 hours