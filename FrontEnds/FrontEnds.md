# FrontEnds architecture

Gridsuite is divided in products that each have their frontend : 

- Grid Admin
- Grid Explore 
- Grid Study
- Grid Monitor
- Grid Dyna

Each frontend share common feature & code that are stored in the [common-ui library ](https://github.com/gridsuite/commons-ui)

To build a new frontend, a template frontend is define in [https://github.com/gridsuite/gridapp-template](https://github.com/gridsuite/gridapp-template)

## Technical Stack


The technical stack of the frontend is based on React and Typescript. The detailed list of dependencies can be found in the [frontend dependencies documentation](FrontEndsDependencies.md)

## Spreadsheet

To by pass the limitation of agGrid community edition, we create a spreadsheet component that extend the agGrid component . More information about the spreadsheet component can be found in the [spreadsheet documentation](https://github.com/gridsuite/gridstudy-app/blob/main/src/components/spreadsheet-view/README.md)


## Powsybl-network-viewer

Gridsuite uses [Powsybl-network-viewer](https://github.com/powsybl/powsybl-network-viewer) to render network area , single line and substation diagrams. 


## Design system

Gridsuite uses a design system to ensure consistency across all frontends. The design system is based on [Material Design](https://material.io/) and is implemented using [Material-UI](https://mui.com/). 


## Supported browsers 

Gridsuite supports the following browsers:
- Google Chrome 
- Mozilla Firefox 
- Edge 





