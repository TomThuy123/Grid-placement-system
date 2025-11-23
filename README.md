# Grid-placement-system
## A roblox placement system
## Note:
### -All of the scripts will be within the rbxl file. The one for showcase will be stated below.
### -This system was extracted from a tycoon game that I made. Hence, you will see some tycoon-related stuff in the scripts.
### -The system is a gridded placement system that supports only for 1 floor. It consists of placement validation within a plot and non-intersection between objects. The showcase also consists of a mobile support system and an inventory system.
### -The system that I am showcasing is the placement system. However, I do use some open-sourced modules from the Roblox devforum, and those modules are not for showcase. They are FastSignal, Packet, 2 modules that were used to help optimize signaling for bindable and remote events. Again, they are not the spotlight of the showcase, they are just there to replace the default roblox signalling which are much slower. I have already asked the RoDev support team and they said that this was allowed, the evidence will be the evidence file. 


## All main scripts that are used as showcase for upgrading the role:
### -Local script and ClientPlacer(StarterPlayerScripts)
### -Server, PlotManager, PlotSpawnPool,PlotStorage(ServerScriptService)
### -MainplaceableManager(ServerScriptService)(Not necessarily needed in the showcase, it is merely a server tracker for all items placed)
### -PlacementValidator(ReplicatedStorage)
## Misc
### -PlaceableObjects(ReplicatedStorage)
### =>Create models for objects that is allowed to be placed. If the model has a submodel under the name of FilterSection or that it has the tag "FilterSection", then that submodel will be the determiner for the object placement validation. Any parts outside of this submodel would be ommitted from placement validation(Can intersect other objects or be outside of player's plot).
