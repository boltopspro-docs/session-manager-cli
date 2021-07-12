<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/session-manager-cli/blob/master/lib/session_manager/help/completion.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

Example:

    session-manager completion

Prints words for TAB auto-completion.

Examples:

    session-manager completion
    session-manager completion hello
    session-manager completion hello name

To enable, TAB auto-completion add the following to your profile:

    eval $(session-manager completion_script)

Auto-completion example usage:

    session-manager [TAB]
    session-manager hello [TAB]
    session-manager hello name [TAB]
    session-manager hello name --[TAB]
