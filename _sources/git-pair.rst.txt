.. post:: Mar 25, 2017
   :tags: emacs, code
   :author: Daniel De Aguiar

==============================
Configure Git Pairs with Magit
==============================

Like me, you've probably used a git pair script at some point. There
are a number of such scripts out there [#f1]_ and they
all work the same way -- creating a local git config with the pair
author information.

While I used to be an avid user of the git cli, that's changed since I
switched to Magit. Unfortunately, I still need to do so when I want to
setup my git pairing session. I decided to change that, partly to keep
me from having to switch out of emacs but also as an exercise to
improve my elisp chops.

Here are my requirements:

1. Only override the commit author, not the committer.
2. A single command to add the author override through the mini-buffer.
3. A single command to remove the author override.
4. A single command to toggle the author override.
5. Persist the override across buffers.
6. The overridden author value must be visible in the Magit commit popup.

After some code spelunking, I found that Magit exposes [#f2]_ the variable
``magit-commit-arguments``, defined in ``git-commit.el``, which holds
a list of strings representing commit arguments. While it is
customizable, my requirements call for a user experience that's little
more dynamic than setting an emacs custom variable. But, by using this
variable, I'll satisfy requirements ``1``, ``5`` and ``6``. Time to
write some code to manipulate it!

First I'll create a function named ``my/git-set-author`` that
let's me add an ``--author`` commit argument.

.. code-block:: elisp

   (defun my/git-set-author (author)
    "Sets the '--author' argument to the input author."
    (when (not (string= "" author))
      (add-to-list 'magit-commit-arguments (concat "--author=" author))
      (minibuffer-message (concat "Author overridden with '" author "'"))))

This works but it's not interactive, meaning it can't be invoked from
the minibuffer, and I won't get prompt for input. It's a good start,
though so I'll add another function that fills in the gaps.

.. code-block:: elisp

   (defun my/git-override-author ()
    "Activates a git commit author override using the input author."
    (interactive)
    (let ((author (read-string "i.e., A U Thor <author@example.com>: "
                  my/magit-gc-override-author)))
      (my/git-set-author author)))

The function ``my/git-override-author`` *is* interactive and prompts
me with a message that's a convenient reminder of the ``--author``
format. This satisfies requirement ``2``. Now that I can set it, I
need to be able to remove it. The function,
``my/git-remove-author-override`` will do that.

.. code-block:: elisp

   (defun my/git-remove-author-override ()
     "Removes the '--author' commit argument."
     (interactive)
     (setq magit-commit-arguments
           (remove-if (lambda (s) (string-match "--author" s))
                      magit-commit-arguments))
     (minibuffer-message "Author override removed."))

Great, now I can add and remove overrides! That checks off requirement
``3``. Time to add override toggling. To do that I'll define the
variable ``my/magit-gc-override-author`` to persist the author
override. I can then update ``my/git-override-author`` to use it as
follows:

.. code-block:: elisp
   :emphasize-lines: 5

   (defun my/git-override-author ()
     "Activates a git commit author override using the input author."
     (interactive)
     (let ((author (read-string "i.e., A U Thor <author@example.com>: " my/magit-gc-override-author)))
       (setq my/magit-gc-override-author author)
       (my/git-set-author author)))

Now I can implement ``my/git-author-toggle`` to check of requirement ``4``.

.. code-block:: elisp

   (defun my/git-author-toggle ()
     "Toggles the git commit author override."
     (interactive)
     (if (find-if (lambda (s) (string-match "--author" s)) magit-commit-arguments)
       (my/git-remove-author-override)
       (my/git-set-author my/magit-gc-override-author)))

Finally, I'll tie it all together through some keybindings that get
set when *Magit* is initialized. I really like ``use-package`` for
things like this.

.. code-block:: elisp

   (use-package magit
     :bind (("C-c C-p" . my/git-override-author)
            ("C-c C-u" . my/git-remove-author-override)
            ("C-c C-t" . my/git-author-toggle)))

   ;; Require 'magit-commit otherwise magit-commit-arguments won't be
   ;; initialized until you first try to commit something.
   (require 'magit-commit)

In the future I'll probably add some niceties like support for
selecting known pairs and support for pair initials but this is good
enough for now.

The full implementation can be found in my emacs `config
<https://github.com/ddeaguiar/dotfiles/blob/e860470f47857d13798ca62f0dd5f9d5347f1872/emacs/.emacs.d/personal/init.el#L580>`_
[#f3]_ [#f4]_. I've got some other goodies in there but, as you peruse my
config, keep in mind that I use the Prelude Emacs distribution.

Cheers!

/Dan

.. rubric:: Footnotes

.. [#f1] Here are a few:
         `pivotal/git_scripts
         <https://github.com/pivotal/git_scripts#git-pair-commit>`_,
         `ryanbriones/git-pair
         <https://github.com/ryanbriones/git-pair>`_,
         `peterjwest/git-pair <https://github.com/peterjwest/git-pair>`_.
.. [#f2] Check out the `source <https://github.com/magit/magit/blob/dc7aa5ca782fd15910ea87b25733f74bde66ecaf/lisp/magit-commit.el#L46>`_.
.. [#f3] I suppose this goes without saying, but **never** lift config without understanding what it does first!
.. [#f4] I'm constantly mucking with my Emacs configuration. It's like a
         virtual garden that requires constant tending.
