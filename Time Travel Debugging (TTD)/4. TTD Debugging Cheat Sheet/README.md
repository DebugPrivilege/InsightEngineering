## Step

| Command | Description                          |
|---------|--------------------------------------|
| `p`       | Step                                 |
| `p-`      | Step backwards                       |
| `ph`      | Step to branch                       |
| `pt`      | Step to return                       |
| `p-t`     | Step backwards to previous return    |

## Trace

| Command     | Description                           |
|-------------|---------------------------------------|
| `t`           | Trace                                 |
| `t-`          | Trace backwards                       |
| `ta <addr>`   | Trace to `<addr>`                     |
| `t-a <addr>`  | Trace backwards to `<addr>`           |

## Go

| Command                              | Description                                               |
|--------------------------------------|-----------------------------------------------------------|
| `g`                                  | Go forward                                                |
| `gt <position>`                      | Go forward to the given time travel position              |
| `g-`                                 | Go backwards                                              |
| `g <foo!bar \| funcAddr>`            | Go until reaching the start of the function `foo!bar`     |
| `g- <foo!bar \| funcAddr>`           | Go backwards until reaching the start of the function `foo!bar` |
| `gt <position>`                      | Go forward and break on `<position>`                      |
| `g-t <position>`                     | Go backwards and break on `<position>`                    |
| `!tt <position>`                     | Jump to a location in the trace                           |

## Breakpoints

| Command                  | Description                              |
|--------------------------|------------------------------------------|
| `bp <symbolic name or address>` | Set a breakpoint                      |
| `bt <position>`          | Set a time/position breakpoint           |

## Others

| Command         | Description                                  |
|-----------------|----------------------------------------------|
| `!position`     | Current position for each thread             |
| `!gle` | Displays the last error code and its description for the current thread |

