Elden Ring Armorset Optimizer
================

## Introduction

<br> I saw this report as an opportunity to prove the R skills that I
picked up this semester and expand upon them. I chose an optimization
problem because I thought that I would come to a satisfying conclusion
after learning how to build a model in R. I picked this dataset and
question because Elden Ring is one of my favorite games of all time. It
is notoriously hard and I felt that if I could come to an answer that
could make the game easier, it would be worth my time.

### Primary Questions

<br> Given the Ultimate Elden Ring dataset, I wanted to figure out the
best possible armor combinations against each of the different damage
types. I also wanted to see how these armor combinations changed when
you changed the player’s weight and roll speed.This was an optimization
problem where I was putting together the **best combination of armor
pieces to minimize damage taken**.

## Data

The data that I am using for this project can be accessed from [Kaggle :
Ultimate Elden Ring with Shadow of the
Erdtree](https://www.kaggle.com/datasets/pedroaltobelli/ultimate-elden-ring-with-shadow-of-the-erdtree-dlc)

The first thing that I did was call in the packages I may need while
wrangling this data

``` r
library(dplyr)
library(tidyr)
```

Then I unzip the file in my Google Drive and bring in the two datasets I
needed from the zip: the armor set and the weapons set. I will be
performing my optimization based on the armor csv and the weapon csv is
needed to add weapon weight into my calculations

``` r
# project_files <- unzip('G:/My Drive/MSBA/MSBR 70280/archive.zip')

armors <- read.csv('C:/Users/liams/OneDrive/Documents/R Practice/er-optimizer/data/armors.csv')
weapons <- read.csv('C:/Users/liams/OneDrive/Documents/R Practice/er-optimizer/data/weapons.csv')
```

For the most part the data was clean, but there was some funky stuff
going on in the damage negation column. Inside of each row, all of the 8
different damage types were held in this column. Before I could do
anything, I needed to break this apart. I put the damage types into a
blank data frame, then binded the damage data frame to the original
armor one.

``` r
armors$damage.negation <- gsub("\\[|\\]|\\{|\\}|'", "", armors$damage.negation)

init_split <- strsplit(armors$damage.negation, ",")

blank_matrix <- matrix(NA, ncol = 8, nrow = length(init_split))

for(i in 1:length(init_split)) {
  blank_matrix[i, ] <- init_split[[i]]
}

blank_matrix <- as.data.frame(blank_matrix)

new_names <- c("phy", "vs_str", "vs_sla", "vs_pie", 
               "mag", "fir", "lit", "hol")

colnames(blank_matrix) <- new_names

blank_matrix[, new_names] <- lapply(new_names, function(x) {
  gsub("^.*: ", "", blank_matrix[, x])
})

armors <- cbind(armors, blank_matrix)
```

Next I do some work in the armor data frame to make it easier to work
with. I converted all of the damage resistances to numeric. I had to
clean up the DLC column into being binary, as before it either had a 0,
1, or it read “Base Game”. Finally, I selected the columns I needed for
analysis.

``` r
armors <- armors %>% 
            mutate(phy = as.numeric(phy),
                   vs_str = as.numeric(vs_str),
                   vs_sla = as.numeric(vs_sla),
                   vs_pie = as.numeric(vs_pie),
                   mag = as.numeric(mag),
                   fir = as.numeric(fir),
                   lit = as.numeric(lit),
                   hol = as.numeric(hol)) %>% 
            mutate(dlc = case_when(dlc == 'Base Game' ~ 0,
                                   dlc == 0 ~ 0,
                                   TRUE ~ 1)) %>% 
            select(1, 2, 5, 8, 12:20)
```

In order to make the optimization run smoother, I created dummy
variables for each different armor piece. This allowed me to keep
everything in one dataframe, as opposed to breaking it out by armor
type.

``` r
library(fastDummies)
armors <- dummy_cols(armors, select_columns = 'type')
names(armors)[names(armors) == 'type_chest armor'] <- 'type_chest'
names(armors)[names(armors) == 'type_leg armor'] <- 'type_leg'
```

The last “wrangling” I need to do is creating some objects that hold
important numbers to my optimization. This means different potential
player weights and our base weapon weight. I chose the base equip load
based on a normal end game endurance level. I chose the weapon that I
did because it was around the average weight of all weapons and it is
something that you would probably be carrying around the end of the
game. I also made the three different equip loads for each different
roll type. Note: this weight does not include any talismans you may be wearing,
just armor and the Rivers of Blood katana.

``` r
base_EL <- 84.1

weapon_weight <- weapons$weight[weapons$weapon_id == 154]

light_EL <- (0.299 * base_EL) - weapon_weight
med_EL <- (0.699 * base_EL) - weapon_weight
heavy_EL <- (0.99 * base_EL) - weapon_weight
```

## Methods

<br> This armor optimization is a **bin packing** problem. I have what I
want to maximize, damage negation, and I have a few constraints that my
result set needs to follow. One constraint is that total weight needs to
be less than the max weight that I set. Another is that there can only
be at most one armor piece per piece type. This is because you cannot
wear two chest pieces or two gauntlets at a time. I also create two
optimization models, one of which has a constraint for only pulling
items from the base game and one that includes items from the DLC. <br>
First, I pull in the packages I need to optimize.

``` r
library(ompr)
library(ompr.roi)
library(ROI.plugin.glpk)
library(magrittr)
```

Next, I create the different variables I need for my optimization
function and for my mapply.

``` r
armor_row <- nrow(armors)

obj_vars <- colnames(armors)[6:13]

el_vars <- c(light_EL, med_EL, heavy_EL)
obj_vars <- colnames(armors)[6:13]

option_combos <- expand.grid(obj_vars = obj_vars, 
                             el_vars = el_vars)

option_combos$obj_vars <- as.character(option_combos$obj_vars)
```

Here is my optimization function with the base game constraint. This is
what I will run through my mapply. This creates a function based on the
restraints I talked about earlier. It then solves the function and
brings it back to the complete data data frame.

``` r
optim_function_base <- function(obj_vars, el_vars) {
  obj_var_loop <- armors[, obj_vars]

  full_model <- MIPModel() %>% 
    add_variable(x[i], i = 1:armor_row, type = 'binary') %>% 
    set_objective(sum_expr(x[i] * obj_var_loop[i], i = 1:armor_row), sense = 'max') %>% 
    add_constraint(sum_expr(armors$weight[i] * x[i], i = 1:armor_row) <= el_vars) %>% 
    add_constraint(sum_expr(armors$type_helm[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_chest[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_gauntlets[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_leg[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$dlc[i] * x[i], i = 1:armor_row) == 0)
  
  model_solve <- solve_model(full_model, with_ROI('glpk'))  
  
  model_solution <- get_solution(model_solve, x[i])
  
  complete_data <- cbind(armors, model_solution)
  
  output <- complete_data[complete_data$value > 0, ]
  output$obj_var <- obj_vars
  output$el_vars <- el_vars
  output
}
```

Next I put this into an mapply. The mapply runs through all the
different possible damage resistances and equip loads and gives me a
nice data frame with all the optimal combinations.

``` r
all_combo_optim_base <- mapply(
  optim_function_base, 
  option_combos$obj_vars[1:24], 
  option_combos$el_vars[1:24], 
  SIMPLIFY = FALSE
  )

all_combo_optim_base <- do.call(rbind, all_combo_optim_base)
```

I then repeat this process without the base game constraint just to
personally see what DLC items can be valuable.

``` r
optim_function_full <- function(obj_vars, el_vars) {
  obj_var_loop <- armors[, obj_vars]

  full_model <- MIPModel() %>% 
    add_variable(x[i], i = 1:armor_row, type = 'binary') %>% 
    set_objective(sum_expr(x[i] * obj_var_loop[i], i = 1:armor_row), sense = 'max') %>% 
    add_constraint(sum_expr(armors$weight[i] * x[i], i = 1:armor_row) <= el_vars) %>% 
    add_constraint(sum_expr(armors$type_helm[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_chest[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_gauntlets[i] * x[i], i = 1:armor_row) <= 1) %>% 
    add_constraint(sum_expr(armors$type_leg[i] * x[i], i = 1:armor_row) <= 1)
  
  model_solve <- solve_model(full_model, with_ROI('glpk'))  
  
  model_solution <- get_solution(model_solve, x[i])
  
  complete_data <- cbind(armors, model_solution)
  
  output <- complete_data[complete_data$value > 0, ]
  output$obj_var <- obj_vars
  output$el_vars <- el_vars
  output
}

all_combo_optim_full <- mapply(
  optim_function_full, 
  option_combos$obj_vars[1:24], 
  option_combos$el_vars[1:24], 
  SIMPLIFY = FALSE
)

all_combo_optim_full <- do.call(rbind, all_combo_optim_full)
```

## Results

<br> I do not have any visualizations, but I do have two data frames
each consisting of 24 different armor combinations. There is a
combination for each damage type (8) and for each equip load (3). While
this was not a statistics problem, I know that my model provided me with
the best possible armor combination to negate a certain type of damage.
The data frames are *all_combo_optim_base* and *all_combo_optim_full*.
The second to last column in the data frames tells you which damage type
it is optimizing based on, and the last column gives you the equip load
constraint that it followed. If you were looking to stop physical damage
at a medium equip load while playing the base game, you would wear the
Bull-Goat Helm, Radahn’s Lion Armor, Banished Knight Gauntlets, and
Bull-Goat Greaves. If you include the DLC, you actually change your helm
and leg armor, instead equipping the GreatJar and the Verdigris Greaves.
Follwoing are the main damage types and equip loads I focused on for my
report. I will show how you can easily find them and what they look like
in game. <br> This is the armor set for light rolling against slashing
damage.

``` r
lr_vs_slash <- all_combo_optim_base %>% 
                  filter(obj_var == 'vs_sla',
                         el_vars == light_EL) %>% 
                  select(name, type, weight, vs_sla)
lr_vs_slash
```

                                   name        type weight vs_sla
    vs_sla.143       Land of Reeds Helm        helm    3.6    4.8
    vs_sla.344 Marionette Soldier Armor chest armor    8.8   13.5
    vs_sla.573      Sorcerer Manchettes   gauntlets    1.1    1.3
    vs_sla.683           Zamor Legwraps   leg armor    5.1    6.8

<figure>
<img
src="https://github.com/therealsmithy/er-optimizer/blob/main/images/e1b99d2f-d2bb-472c-94dd-6df002fae08d.jpg"
alt="Optimal Armor for Light Rolling Against Slashing Damage" />
<figcaption aria-hidden="true">Optimal Armor for Light Rolling Against
Slashing Damage</figcaption>
</figure>

<br> Here is the armor set for medium rolling against magic damage.

``` r
mr_vs_mag <- all_combo_optim_base %>% 
                filter(obj_var == 'mag',
                       el_vars == med_EL) %>% 
                select(name, type, weight, mag)
mr_vs_mag
```

                               name        type weight  mag
    mag.1631         Nox Mirrorhelm        helm    7.5  6.7
    mag.2621 Azur's Glintstone Robe chest armor    7.1 15.4
    mag.561      Preceptor's Gloves   gauntlets    2.1  3.6
    mag.663    Preceptor's Trousers   leg armor    3.9  8.3

<figure>
<img
src="https://github.com/therealsmithy/er-optimizer/blob/main/images/5dad67e0-41be-4f68-bf22-3fcc1748d41b.PNG"
alt="Optimal Armor for Medium Rolling Against Magic Damage" />
<figcaption aria-hidden="true">Optimal Armor for Medium Rolling Against
Magic Damage</figcaption>
</figure>

<br> And here is the armor set for heavy rolling against holy damage,
allowing items from the DLC.

``` r
hr_vs_hol <- all_combo_optim_full %>% 
                filter(obj_var == 'hol',
                       el_vars == heavy_EL) %>% 
                select(name, type, weight, hol)
hr_vs_hol
```

                                  name        type weight  hol
    hol.1182                 Greathood        helm    5.1  6.2
    hol.2831       Crucible Tree Armor chest armor   15.5 14.5
    hol.5302 Godskin Apostle Bracelets   gauntlets    2.1  3.6
    hol.5872        Dryleaf Cuissardes   leg armor    3.1  8.1

<figure>
<img
src="https://github.com/therealsmithy/er-optimizer/blob/main/images/059182d5-b899-4bda-8e33-96e1dd28da06.PNG"
alt="Optimal Armor for Heavy Rolling Against Holy Damage" />
<figcaption aria-hidden="true">Optimal Armor for Heavy Rolling Against
Holy Damage</figcaption>
</figure>

## Discussion

<br> When playing a video game that takes some over 100 hours to
complete in full, there are plenty of decisions to be made. If you are
making these decisions without any guidance, the choices you make may
make your life a lot harder. I think that my findings are interesting
and significant because they can be applied to pretty much any situation
the game puts you in to. As long as you are able to find the armor it
recommends, you can set yourself up to dominate any encounter if you
know what type of damage type you will be facing.
