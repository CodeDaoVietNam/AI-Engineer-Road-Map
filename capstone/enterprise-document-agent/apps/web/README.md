# Web application boundary

`web` owns the React reviewer dashboard and user-interface concerns only. It
consumes published HTTP API contracts; it must not import Python packages,
implement domain rules, or access infrastructure services directly.

Features belong here only when they are client-side presentation, interaction,
or client state. Authorization and business decisions remain on the server.
