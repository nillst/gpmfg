

```python
from gpkit import Model, Variable, units, parse_variables
# from jho import aircraft
```

## Generic Multi-Ply layup process

Creates a generic multi-ply layup process using ACCEM equations. Recall the units for the ACCEM models are inches for length and hours for time. The generic multi-ply layup (similar to what Mike and the JHO team actually did)

  * Cut plies from roll bulk material
  * Saturate the plies with resin and then squegee excess
  * Place ply on stack on mold
  * Place peel-ply, breather, bag (and sometimes mylar to get a nice surface finish)
  * Cure (won't count toward labor time)
  * Remove bag, breather, flow medium
  * Trim OML

### Process models adapted from ACCEM and COSTADE

#### Cut plies from bulk material
Adapted from ACCEM equation F2

$$ t = 0.05 + n_\mathrm{plys}(0.0015 L_{trim}) $$

| <td colspan=3> Geometry Variables |
| -------------------------------- |
| Variable | Description | Units |
| $n_\mathrm{plys}$ | Number of plys in the stack | $\text{count}$ |
| $L_\mathrm{trim}$ | The perimeter of the ply to cut | $\text{in}$ |

#### Impregnate with dry ply with resin
COSTADE 0040


Explaination: Apply and smooth liquid resinwith a putty knife. Application is to both faces of an interface.


EquationID : 14

$$ t [\text{minutes}] = 5 + n_\mathrm{plys} \left(1 + \frac{A}{40} \right) $$

| <td colspan=3> Geometry Variables |
| --------------------- |
| Variable | Description | Units |
| A | ply area | $\text{in}^2$ |
| $n_\mathrm{plys}$ | Ply count | $\text{count}$ |

#### Layup plys directly on tool
ACCEM equation L6

$$ t [\text{minutes}] = 0.05 + 0.000751 n_\mathrm{plys} A_\mathrm{layup}^{0.6295} $$

| <td colspan=3> Geometry Variables |
| --------------------- |
| Variable | Description | Units |
| A | Layup area | $\text{in}^2$ |
| $n_\mathrm{plys}$ | Ply count | $\text{count}$ |

#### Net Trim Cured Part
ACCEM equation F2

$$ t = 3[\text{minutes}] + \frac{L_{trim}} {11}  $$

| <td colspan=3> Geometry Variables |
| -------------------------------- |
| Variable | Description | Units |
| $n_\mathrm{plys}$ | Number of plys in the stack | $\text{count}$ |
| $L_\mathrm{trim}$ | The perimeter of the layup to trim | $\text{in}$ |

### GPkit Model Implementation


```python
# create the generic multi-layup process
from gpkit import Model, Variable, parse_variables

class MultiLayup(Model):
    '''Generic process for creating a multi-ply composite part.
    Includes processes for:
        - Manually cutting plys from bulk material
        - Impregnating the dry charges with resin
        - Lay up the impregnated parts onto the tool
    
    Process Time Variables
    ----------------------
    t          [hrs]    Total part process time
    tcut      [hrs]    Process time to cut the charges from broadgoods
    tresin    [hrs]    Process time for putting resin into the 
    tlayup    [min]    Process time to layup the impregnated charges
    ttrim     [min]    Process time to trim cured part
    
    Geometry Variables
    ------------------
    numplys      [count]    Number of plys in the layup
    perimeter    [in]       Perimeter of the layup
    area         [in^2]     Layup area
    '''
    def setup(self):
        exec parse_variables(self.__doc__)
        
#         numplys = self.numplys = Variable('n_\\mathrm{plys}','count','Number of plys in the layup')
#         perimeter = self.perimeter = Variable('l_\\mathrm{perimeter}','in','Perimeter length of the layup')
#         area = self.area = Variable('A_\\marthrm{layup}','in^2','Area of the layup')
        
        constraints = [
            t >= tcut + tresin + tlayup,
            tcut >= 0.05*units('hrs') + numplys*(0.0015*units('hrs/in')*perimeter),                         # Cut out plys
            tresin >= 5*units('min') + numplys*(1*units('min/count') + area/(40*units('in^2/min'))),        # Impregnate with resin
            tlayup >= 3*units('min') + numplys*(units('hrs/count')*((area/units('in^2'))**0.6295/1331)),    # Layup Plies            
        ]
        return constraints
```

## The Factory contains all the processes and calculates the labor rate


```python
class JHOFactory(Model):
    '''evaluates the unit labor cost of manufacturing the input jho aircraft
    
    Input Variables
    ---------------
    plabor    100    [USD/hr]    labor cost rate
    
    Labor Variables
    ---------------
    tlabor    [hr]    total labor time
    
    Cost Variables
    --------------
    clabor    [USD]    total cost of labor
    
    Geometry Variables
    ------------------
    winga       [in^2]     Area of wing layers
    wingp       [in]       Perimiter of wing ply stack
    wingplys    [count]    Count of wing plys
    fuesa       [in^2]     Area of fuselage
    fuesp       [in]       Perimiter of fuselage ply stack
    fuesplys    [count]    Count of plys in the fuselage stack

        
    '''
    def setup(self):
        exec parse_variables(self.__doc__)
        processes = self.processes = dict()
        processes['wing'] = MultiLayup()
        processes['fuselage'] = MultiLayup()
        
        constraints = [
            tlabor >= sum([p.t for p in processes.values()]),
            clabor >= tlabor*plabor
        ]
        return self, constraints
    
#     def setup(self):
# #         exec parse_variables(self.__doc__)
#         processes = self.processes = dict()
#         processes['wing'] = MultiLayup()
#         processes['fuselage'] = MultiLayup()
        
#         constraints=[]
        
# #         constraints = [
# #             tlabor >= sum([p.t for p in processes.values()]),
# #             clabor >= plabor*tlabor
# #         ]
        
#         return self, processes, constraints
```
