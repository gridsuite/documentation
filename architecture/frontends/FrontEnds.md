# FrontEnds architecture

Gridsuite is divided in products that each have their frontend :

- Grid Admin
- Grid Dyna
- Grid Explore
- Grid Monitor
- Grid RT
- Grid Study

Each frontend share common feature & code that are stored in the [commons-ui library](https://github.com/gridsuite/commons-ui)

To build a new frontend, a template frontend is define in [https://github.com/gridsuite/gridapp-template](https://github.com/gridsuite/gridapp-template)

## Technical Stack

The technical stack of the frontend is based on React and TypeScript.

It relies on the following key dependencies (for simplicity, only the main ones are listed here):

- `@hello-pangea/dnd` : accessible drag-and-drop
- `@mui/material` : Material Design UI components (MUI)
- `@tanstack/react-virtual` : list and table virtualization
- `@xyflow/react` : interactive diagram/flow editor (used for network area diagrams)
- `ag-grid-community` : advanced data grid
- `mapbox-gl` / `maplibre-gl` / `react-map-gl` : map rendering (geo data visualization)
- `oidc-client-ts` : OpenID Connect / OAuth2 authentication
- `plotly.js` / `react-plotly.js` : interactive charting
- `react-hook-form` : form management
- `react-intl` : internationalization (i18n) for React (FormatJS)
- `react-router` : client-side routing
- `redux` : state management
- `yup` : form validation


## Powsybl-network-viewer

Gridsuite uses [Powsybl-network-viewer](https://github.com/powsybl/powsybl-network-viewer) to render network area , single line and substation diagrams.

## Design system

Gridsuite uses a design system to ensure consistency across all frontends. The design system is based on [Material Design](https://material.io/) and is implemented using [Material-UI](https://mui.com/).

## Spreadsheet

To bypass the limitations of the AG Grid Community Edition, we created a spreadsheet component that extends AG Grid. More information about the spreadsheet component can be found in the [spreadsheet documentation](https://github.com/gridsuite/gridstudy-app/blob/main/src/components/spreadsheet-view/README.md)

## Table components: AG Grid vs Tanstack Table

AG Grid Community is the default and standard solution across Gridsuite frontends for displaying tabular data. However, TanStack Table can also be used. For more information, see [AG Grid and TanStack Table](AGGrid_TanstackTable.md).

## Supported browsers

Gridsuite is regulary tested with the following browsers:
- Google Chrome
- Mozilla Firefox
- Edge
