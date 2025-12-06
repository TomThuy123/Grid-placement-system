
local ClientPlacer = {}
ClientPlacer.__index = ClientPlacer

--//Services
local RP=game:GetService("ReplicatedStorage") --Get access to Replicated Storage
local UIS=game:GetService("UserInputService") --Access to keyboard, mouse, input elements and commands, etc... 
local RunService=game:GetService("RunService") --Get access to Runservice, which will allow me to use BindToRenderStep in this code
local ContextActionService = game:GetService("ContextActionService") --Get access to ContextActionService, which helps to bind input actions to functions
local TweenService=game:GetService("TweenService") --Get access to TweenService, allowing the usage of TweenService:GetValue to convert 0->1 values in linear form to other EasingStyle and EasingDirection form. This is literally the only usage of TweenService in this code.
--//Modules
local PlacementValidator=require(RP.PlacementValidator)	
local FastSignal=require(RP.Modules.Libraries.FastSignal) --Get access to FastSignal, an open-source library on Roblox Devforum as a singalling library. I use this because it is better than roblox Bindable Events in terms of communicating between client-sided scripts
local Packet=require(RP.Modules.Libraries.Packet) --Get access to Packet, an open-source library on Roblox Devforum as a networking library. It is better than using remote events in terms of speed, so I use it.
--//PlayerGUI
local PlayerGUI=game.Players.LocalPlayer.PlayerGui --Get access to PlayerGui
--//Misc
local boxOutlineTemplate = RP.BoxOutline --Refers to a SelectionBox instance, it helps visualization when placing stuff

local DeleterBoxOutline = boxOutlineTemplate:Clone() --Clones the template to create the Outlines for when items are supposed to be deleted
DeleterBoxOutline.Color3=Color3.fromRGB(255, 0, 0) --Changes color to rgb value of 255, 0, 0
DeleterBoxOutline.LineThickness=0.05 --Changes the SelectionBox outline thickness to 0.05 
DeleterBoxOutline.SurfaceColor3=Color3.fromRGB(255,0,0) --Changes color to rgb value of 255, 0, 0

local PlaceableObjects=RP:WaitForChild("PlaceableObjects") --Get access to the folder in Replicated Storage that contains all the models that can be placed
--Camera
local Camera=workspace.CurrentCamera --Get access to camera
--//Players
local plr=game.Players.LocalPlayer --Get access to plr instance
local Char=plr.Character or plr.CharacterAdded:Wait() --Get access to the player's character
--//Context variables
local PREVIEW_RENDER="RenderPreview" --Context for rendering previews of models when selected for placement
local PLACE_ACTION="Place" --Context for placing action
local ROTATE_ACTION="Rotate" --Context for rotating action
local DELETE_ACTION="Delete" --Context for deleting
local CYCLE_ACTION="Cycle" --Context for cycling through different placeable items
local SNAP_ACTION = "Snap" --Context for toggling snap to grid mode on and off
local CYCLE_GRID="GridCycle" --Context for cycling through different snap to grid mode, like snap to 1,2,0.125,0.25,0.5 in terms of stud
--//Other variables
local PlaceDebounce=false --Debounce for placing
local DeleteDebounce=false --Debounce for rotating
local OnMobile=false --Variable to check if player is on mobile or not
local InventoryItemCounter={} --A table that later will read for every items within player's inventory, and then count them. It is like a dictionary
local Excluder={Char}  --An excluder table of stuff that will be avoided by raycast when player is in Placement mode
local DeleteExcluder={Char} --An exluder table of stuff that will be avoided by raycast when player is in Delete mode



--Data
local Data --Initialize variable for Data
repeat Data=Packet.RetriveData:Fire() until Data~=nil --Calls to server to retrieve player's Data from the Datastore. This is so the script can read the amount of placeable items the player has in their inventory. This script will not stop calling until data is retrieved
for _, v in pairs(Data.Inventory.PlaceableItems) do --Loop through player's inventory Data
	InventoryItemCounter[v.Name]=v.Amount or 0 --Intialize the item count. If the item exist within Data, set it to its respective amount. If it doesn't exist, then set it to 0
end
coroutine.wrap(function()
	--Basically a background listener. Every time the player's data is updated within the server, the server will send a the new Data to a main client local script.
	--This client local script will then send a signal to other local scripts. This is more efficient than having the server send a signal to every other local script
	--Using coroutine.wrap() here means this connection is created without blocking other functions
	--We basically rebuild the item count table every tim the data is updated
	FastSignal.UpdateData:Connect(function(NewData) 
		Data=NewData --Set the new Data
		for i,v in Data.Inventory.PlaceableItems do
			InventoryItemCounter[v.Name]=(v.Amount or 0) --Rebuild the counter ye
		end
	end)
end)()

--//Device
--Basically decide if the player is on a mobile device or a pc, no controller though sorry lol
--If the device only has touch sensors (no keyboard or mouse), then it is a mobile.
--If it has keyboard and mouse, which means no touch sensors, then it is a pc
--This is needed because the placement system will use different raycast method for ease of use for the players.
--For spoilers, on pc, the item preview will be based on player's mouse position, but on mobile, it will be based on the center of the screen
if UIS.TouchEnabled and not UIS.KeyboardEnabled and not UIS.MouseEnabled then 
	OnMobile=true
elseif not UIS.TouchEnabled and UIS.KeyboardEnabled and UIS.MouseEnabled then
	OnMobile=false
end

--//Functions
--Snap a world position to the placement grid.
--Rounds the X and Z coordinates to the nearest multiple of gridSize, which is a stud number, and keep Y unchanged.
local function snapToGrid(pos, gridSize)
	return Vector3.new(
		math.round(pos.X/gridSize)*gridSize,
		pos.Y,
		math.round(pos.Z/gridSize)*gridSize
	)
end


local function castMouse(TheExcluder) --TheExcluder is the Excluder table that will be used for the raycast
	if OnMobile==false then --If player is not on mobile device, which means they are on pc
		local mouseLocation=UIS:GetMouseLocation() --Get the mouse cursor postion, which is a Vector2
		local ray=Camera:ViewportPointToRay(mouseLocation.X,mouseLocation.Y) --Change the Vector2 position into a 3D ray from the current camera.

		local raycastParams = RaycastParams.new() --Set up a new raycast parameter
		raycastParams.FilterType = Enum.RaycastFilterType.Exclude --Change the mode to Filter to avoid items in the excluder table
		raycastParams.FilterDescendantsInstances = TheExcluder --Set the filter table

		return workspace:Raycast(ray.Origin,ray.Direction*1000,raycastParams) --Do raycast from the ray origin, which go out by 1000 studs.
	else --If player is on mobile
		local vp=Camera.ViewportSize --Get the mobile device screen size
		local center=Vector2.new(vp.X*0.5,vp.Y*0.4) --Set a Vector2 that is horizontally centered but will be raised a bit vertically instead of directly being in the middle
		                                            --Based on testing, I find it looked better if it was 0.4 instead of 0.5

		local ray=Camera:ViewportPointToRay(center.X, center.Y) --Change the Vector2 position into a 3D ray.

		local params = RaycastParams.new() --Set up a new raycast parameter
		params.FilterType = Enum.RaycastFilterType.Exclude --Change the mode to Filter to avoid items in the excluder table
		params.FilterDescendantsInstances = TheExcluder --Set the filter table

		return workspace:Raycast(ray.Origin, ray.Direction * 1000, params) --Do raycast from the ray origin, which go out by 1000 studs.
	end
end


--Functions to add/remove objects into the excluder tables
function ClientPlacer.AddExcluder(object)
	table.insert(Excluder,object)	
end
function ClientPlacer.RemoveExcluder(object)
	table.remove(Excluder,table.find(Excluder,object))
end

--Functions to enable or disable visual mode.
--For spoilers, when player is in Placement mode, all objects will have white outlines to indicate hitbox, and conveyor parts will have visible moving arrow lines.
--When player is in delete mode, all of the visual above will be invinsible.
-- The colon ":" makes this a method, so the first hidden parameter is `self`, allowing the function to be called from outside the modulescript
-- Self is an instance created by setmetatable in ClientPlacer.new().
function ClientPlacer:EnableVisualBorder(State)
	if not self.Plot then return end --Check if player has plot, if not, the function won't continue and stops
	for i,v in pairs(self.Plot.Objects:GetChildren()) do --Loop through all placed objects within the player's plot
		v.VisualOutline.Visible=State --Change the visibility of the visual border
		if v:HasTag("IsConveyor") then --If the object has the tag "IsConveyor", then it is considered a conveyor object
			for _,j in pairs(v:GetDescendants()) do --Loop through all parts that make up the conveyor
				if j:HasTag("ConveyorPart") then --If the part has the tag "ConveyorPart", then it is a part that can move things
					j.Beam.Enabled=State --Change the visibility of the arrow lines
					--It does not break here because some Conveyor models have multiple parts that is considered a "ConveyorPart"
				end
			end
		end
	end
end
function ClientPlacer.new(plot: Model, ItemName: string)
	-- Create the instance table and give it the ClientPlacer metatable so : methods work.
	local self = setmetatable({
		Plot=plot,	
		PlotItem={},
		Preview=nil,
		PreviewIndex=1,
		Rotation = 0,
		TargetRotation=0,
		GridSize = 1,
		GridSlot=1;
		GridLst={1,2,0.125,0.25,0.5},
		Placeable=false,
		CurrentCF = CFrame.new(),
		TargetPivot=nil,
		Mode="Place"
	},ClientPlacer)
	
	-- This resets the default mode and debounces
	self.Mode="Place"
	PlaceDebounce=false
	DeleteDebounce=false
	
	
	PlayerGUI.MobileDeleteUI.ImageLabel.Visible=false --On mobile, when player is in delete mode, whatever object in the middle of the screen is aimed to be deleted
													  --There will be a Icon to indicate deletion on mobile, for now, set that icon to be invisible
	--Since player is in Placing mode, set the deleting BoxOutline subject to nothing
	DeleterBoxOutline.Adornee=nil 
	DeleterBoxOutline.Parent=nil
	
	
	local objects=Data.Inventory.PlaceableItems --Get access to the player full inventory, the one with the total count of every placeable object they own
	for i,v in pairs(self.Plot.Objects:GetChildren()) do --Loop through all the children of the plot's Object folder, which is basically the folder that have the objects that player have placed
		if not table.find(Excluder,v) then --If the object placed within the plot is not in the Excluder table
			ClientPlacer.AddExcluder(v)--It will be added into the Excluder folder
		end
	end
	
	self:EnableVisualBorder(true) --Enable all visual borders of all placed objects within plot. This makes it easier to see the hitboxes of placed stuff
	
	--Basic UI 
	PlayerGUI.ScreenGui.Snap.Text="Snap: "..'1'.. " stud" --Change the Snap indicator TextLabel to default 1 stud value
	PlayerGUI.ScreenGui.Enabled=true --Enable the ScreenGui of the Snap UI
	PlayerGUI.PlacementView.Main.Visible=true --Make PlacementView UI visible, it has keybind directions like press B to cancel, X to toggle delete mode, etc...
	
	--Check if player entered build mode by manually selecting an item from inventory, or they just press B to enter build mode
	if ItemName then --If they press B to enter build mode, then ItemName will be nil, so this will not continue
		while objects[self.PreviewIndex].Name~=ItemName do --If it does continue, increase the table index until it finds the exact selected object name 
			self.PreviewIndex+=1
		end
	end
	
	
	--Loop through Object folder of the plot, and make a table that counts the items that has been placed
	for i,v in pairs(self.Plot.Objects:GetChildren()) do
		self.PlotItem[v.Name]=(self.PlotItem[v.Name] or 0) + 1; --Count the number of each object placed, if the table dont have the object then it will be set to 0, if it does, get its count value, then add 1 to the count
	end
	
	-- Function to call the preview system
	self:InitiateRenderPreview()
	-- Bind all input actions to their functions with keybinds
	-- Here, the falses make it so that no mobile GUI will be created
	-- and the ... allow the function to take in any amount of arguments, and it will bring the function(...) thing into the self function themselves
	-- The binded action here is binded to a self function
	-- Here the self function are set to metatables so they can be called outside the module script, which is needed so that mobile users can trigger these functions
	-- because they dont have keyboard keys
	-- Here, doing left click will attempt to place object if player is in delete mode, and attempt to delete if player is in delete mode
	-- pressing R will call to RotateBlock function, which ofc rotates the preview object
	-- pressing X toggle delete mode
	-- Pressing G toggle snap mode
	-- Pressing E or Q will cycle the object, E will cycle to the right, and Q will cycle to the left object in player's inventory
	-- Pressing C will cycle the grid size
	ContextActionService:BindAction(PLACE_ACTION, function(...) self:TryPlaceBlock(...) end, false, Enum.UserInputType.MouseButton1)
	ContextActionService:BindAction(ROTATE_ACTION, function(...) self:RotateBlock(...) end, false, Enum.KeyCode.R)
	ContextActionService:BindAction(DELETE_ACTION, function(...) self:TryDelete(...) end, false, Enum.KeyCode.X)
	ContextActionService:BindAction(SNAP_ACTION, function(...) self:ToggleGrid(...) end, false, Enum.KeyCode.G)
	ContextActionService:BindAction(CYCLE_ACTION, function(...) self:CycleObject(...) end, false, Enum.KeyCode.E,Enum.KeyCode.Q)
	ContextActionService:BindAction(CYCLE_GRID, function(...) self:CycleGrid(...) end, false,Enum.KeyCode.C)
	return self --Return self so the local script that calls ClientPlacer.new() get accesss to methods intialized within ClientPlacer, this is so it can do functions
	            --like delete, palce, rotate without having to be directly in this module script, which is needed for mobile users.
end

function ClientPlacer:NewSpecificItem(ItemName)
	local objects = Data.Inventory.PlaceableItems
	if ItemName then
		self.Mode="Place"
		PlayerGUI.MobileDeleteUI.ImageLabel.Visible=false
		DeleterBoxOutline.Adornee=nil
		DeleterBoxOutline.Parent=nil
		self.PreviewIndex=1
		while objects[self.PreviewIndex].Name~=ItemName do
			self.PreviewIndex+=1
		end
		self:PreparePreviewModel(PlaceableObjects[objects[self.PreviewIndex].Name])
	end
end


function ClientPlacer:InitiateRenderPreview()
	local objects = Data.Inventory.PlaceableItems
	self:PreparePreviewModel(PlaceableObjects[objects[self.PreviewIndex].Name])
	RunService:BindToRenderStep(PREVIEW_RENDER, Enum.RenderPriority.Camera.Value, function(...) self:RenderPreview(...) end)
end	

function ClientPlacer:PreparePreviewModel(model: Model)
	if self.Preview then
		self.Preview:Destroy()
	end
	
	self.Preview = model:Clone()
	local boxOutline = boxOutlineTemplate:Clone()
	local Found=false
	for i,v in pairs(self.Preview:GetDescendants()) do
		if v:FindFirstChild("FilterSection") or v:HasTag("FilterSection") then
			boxOutline.Adornee=v
			Found=true
			break
		end
	end
	if Found==false then
		boxOutline.Adornee=self.Preview
	end
	boxOutline.Parent=self.Preview
	for _, part in self.Preview:GetDescendants() do
		if part:IsA("BasePart") then
			part.CanCollide = false
			part.CanQuery = false
			if part:HasTag("BuildModeInvinsible") then
				part.Transparency = 1
			else
				part.Transparency = 0.5
			end
		end
	end

	self.Preview.Parent = workspace
end

function ClientPlacer:TryPlaceBlock(_,state,_)
	if self.Mode=="Place" then
		if state ~= Enum.UserInputState.Begin then
			return
		end
		if self.Preview and PlaceDebounce==false then
			PlaceDebounce=true
			DeleteDebounce=true
			local success,Object=Packet.TryPlace:Fire(self.Preview.Name, self.TargetPivot,self.Placeable, self.Preview)
			if success==true then
				if not self.Preview then return end
				
				
				ClientPlacer.AddExcluder(Object)
				self.PlotItem[self.Preview.Name]=(self.PlotItem[self.Preview.Name] or 0) + 1;
				PlaceDebounce=false
				DeleteDebounce=false
				
				--//Effects
				script.RoPhysics_V2_Stone_06:Play()
				
				local VisualBorder=RP.VisualOutline:Clone()
				VisualBorder.Parent=Object
				VisualBorder.Visible=true
				for _,v in pairs(Object:GetDescendants()) do
					if v:HasTag("FilterSection") or v.Name=="FilterSection" then
						VisualBorder.Adornee=v
						break
					end
				end

				if Object:HasTag("IsConveyor") then
					for _,j in pairs(Object:GetDescendants()) do
						if j:HasTag("ConveyorPart") then
							j.Beam.Color=ColorSequence.new(Color3.new(1,1,1),Color3.new(1,1,1))
							j.Beam.Enabled=true
						end
					end
				end

			else
				DeleteDebounce=false
				PlaceDebounce=false
			end
		end
	else
		if state ~= Enum.UserInputState.Begin then
			return
		end
		local cast=castMouse(DeleteExcluder)
		if cast and cast.Instance then
			if DeleteDebounce==false then
				DeleteDebounce=true
				PlaceDebounce=true
				local success,ObjectName,ActualObject=Packet.TryDelete:Fire(cast.Instance)
				if success then
					ClientPlacer.RemoveExcluder(ActualObject)
					self.PlotItem[ObjectName]=(self.PlotItem[ObjectName] or 1) - 1;
					script.Roblox_UI_Delete:Play()
					DeleteDebounce=false
					PlaceDebounce=false
				else
					DeleteDebounce=false
					PlaceDebounce=false
				end
			end
		end
	end
end



function ClientPlacer:RotateTween()
	local elapsed=0
	local duration=0.3
	local original=self.Rotation
	local diff=self.TargetRotation-original

	repeat
		local alpha = math.clamp(elapsed/duration,0,1)
		local eased = TweenService:GetValue(alpha, Enum.EasingStyle.Back, Enum.EasingDirection.InOut)

		self.Rotation = original + diff * eased

		local dt = task.wait()
		elapsed += dt
	until elapsed >= duration

	self.Rotation = self.TargetRotation
end


function ClientPlacer:RotateBlock(_, state, _)
	if state == Enum.UserInputState.Begin then
		if self.Mode=="Delete" then return end
		self.TargetRotation+=math.pi/2;
		coroutine.wrap(function()
			self:RotateTween()
		end)()
	end
end

function ClientPlacer:TryDelete(_,state,_)
	if state == Enum.UserInputState.Begin then
		if self.Mode=="Place" then
			if OnMobile==true then
				PlayerGUI.MobileDeleteUI.ImageLabel.Visible=true
			else
				PlayerGUI.MobileDeleteUI.ImageLabel.Visible=false
			end
			self.Mode="Delete"
			self:EnableVisualBorder(false)
		else
			PlayerGUI.MobileDeleteUI.ImageLabel.Visible=false
			self.Mode="Place"
			self:EnableVisualBorder(true)
		end
	end
end

local FilterCache = {} 
local function getFilterData(modelName: string)
	local cached = FilterCache[modelName]
	if cached then
		return cached
	end

	local template = PlaceableObjects:FindFirstChild(modelName)
	if not template or not template:IsA("Model") then
		warn("No template model for placeable:", modelName)
		return nil
	end

	local filter
	for _, d in ipairs(template:GetDescendants()) do
		if d:HasTag("FilterSection") or d.Name == "FilterSection" then
			filter = d
			break
		end
	end

	if not filter then
		warn("No FilterSection in template for:", modelName)
		return nil
	end

	local rootCF = template:GetPivot()
	local filterCF, filterSize = filter:GetBoundingBox()
	local offset = rootCF:ToObjectSpace(filterCF)

	local data = {
		size = filterSize,
		offset = offset,
	}
	FilterCache[modelName] = data
	return data
end

function ClientPlacer:RenderPreview()
	if self.Mode=="Delete" then
		if self.Preview~=nil then
			self.Preview:Destroy()
			self.Preview=nil
		end
		PlayerGUI.PlacementView.Main.Viewer.Amount.TextLabel.Text="Delete Mode"
		local cast=castMouse(DeleteExcluder)
		if cast and cast.Instance then
			if not cast.Instance:IsDescendantOf(self.Plot.Objects) then 
				PlayerGUI.PlacementView.Main.Viewer.ItemName.TextLabel.Text="N/A"
				DeleterBoxOutline.Adornee=nil
				DeleterBoxOutline.Parent=nil
				return 
			end
			coroutine.wrap(function()
				local ActualObject=cast.Instance
				while ActualObject.Parent~=self.Plot.Objects do
					ActualObject=ActualObject:FindFirstAncestorWhichIsA("Model")
				end	
				PlayerGUI.PlacementView.Main.Viewer.ItemName.TextLabel.Text=ActualObject.Name
				DeleterBoxOutline.Adornee = ActualObject
				DeleterBoxOutline.Parent = ActualObject
				
			end)()
		end
	else
		if self.Preview==nil then
			DeleterBoxOutline.Adornee=nil
			DeleterBoxOutline.Parent=nil
			local objects = Data.Inventory.PlaceableItems
			self:PreparePreviewModel(PlaceableObjects[objects[self.PreviewIndex].Name])
			
		end
		
		local cast = castMouse(Excluder,self.Plot)
		if cast and cast.Position and cast.Instance then
			local targetPos = if self.GridSize > 0 then snapToGrid(cast.Position, self.GridSize) else cast.Position
			local targetCF = CFrame.new(targetPos) * CFrame.Angles(0, self.Rotation, 0)
			self.TargetPivot=CFrame.new(targetPos) * CFrame.Angles(0, self.TargetRotation, 0)
			local alpha = 0.15
			local newPos = self.CurrentCF.Position:Lerp(targetPos, alpha)
			local velocity = targetPos - self.CurrentCF.Position
			local leanStrength = 0.05
			local tilt = CFrame.new()

			if velocity.Magnitude > 0.001 then
				local localVel = targetCF:VectorToObjectSpace(velocity)
				local tiltX =  -localVel.Z * leanStrength
				local tiltZ =  localVel.X * leanStrength

				tilt = CFrame.Angles(tiltX, 0, tiltZ)
			end
			self.CurrentCF = CFrame.new(newPos) * targetCF.Rotation * tilt
			self.Preview:PivotTo(self.CurrentCF)


			local size 
			local pos
			if self.Preview:HasTag("FilterSection") then
				size=self.Preview:GetExtentsSize()
			else
				for _,v in pairs(self.Preview:GetDescendants()) do
					if v:HasTag("FilterSection") or v.Name=="FilterSection" then
						size=v:GetExtentsSize() 
						if v:IsA("Model") and self.Preview~="Pillar" then
							pos=self.TargetPivot
						else
							pos=self.TargetPivot
						end
					end
				end
			end
			
				
			--//Conveyor
			if self.Preview:HasTag("IsConveyor") then
				for i,v in pairs(self.Preview:GetDescendants()) do
					if v:HasTag("ConveyorPart") then
						v.Beam.Enabled=true
					end
				end
			end
			--//CheckQuantity
			local InventoryAmount=InventoryItemCounter[self.Preview.Name] or 0
			local UsedAmount=self.PlotItem[self.Preview.Name] or 0
			local QuantityCheck=false
			if InventoryAmount>0 and UsedAmount>=0 then
				if InventoryAmount-UsedAmount>0 then
					QuantityCheck=true
				end
			end
			if self.Preview.Name=="Pillar" then
				pos=self.TargetPivot
			end
			
			local filterData = getFilterData(self.Preview.Name)
			if not filterData then
				return 
			end
			local worldFilterCF = self.TargetPivot * filterData.offset

					
			PlayerGUI.PlacementView.Main.Viewer.Amount.TextLabel.Text=tostring(InventoryAmount-UsedAmount).." Items Remaining"
			PlayerGUI.PlacementView.Main.Viewer.ItemName.TextLabel.Text=self.Preview.Name
			if PlacementValidator.WithinBounds(self.Plot, filterData.size, worldFilterCF)
				and PlacementValidator.NotIntersectingObjects(self.Plot, filterData.size , worldFilterCF)
				and cast.Instance:HasTag("IsPlot") and QuantityCheck==true then
				self.Placeable = true
				self.Preview.BoxOutline.Color3 = Color3.new(0, 0.666667, 1)
				self.Preview.BoxOutline.SurfaceColor3=Color3.new(0, 0.666667, 1)
			else
				self.Placeable = false
				self.Preview.BoxOutline.Color3 = Color3.new(1, 0, 0)
				self.Preview.BoxOutline.SurfaceColor3=Color3.new(1,0,0)
			end
		end
	end
	
end

function ClientPlacer:CycleObject(_,state,obj, Mobile)
	if state==Enum.UserInputState.Begin then
		if self.Mode=="Delete" then return end
		local objects = Data.Inventory.PlaceableItems
		local direction
		if not Mobile then
			direction= if obj.KeyCode == Enum.KeyCode.E then 1 else -1
		else
			direction= if Mobile == "E" then 1 else -1
		end
		 
		self.PreviewIndex = (self.PreviewIndex-1+direction)%#objects+1
		self:PreparePreviewModel(PlaceableObjects[objects[self.PreviewIndex].Name])
	end
end

function ClientPlacer:ToggleGrid(_, state, _)
	if state == Enum.UserInputState.Begin then 
		if self.Mode=="Delete" then return end
		self.GridSize = if self.GridSize == 0 then self.GridLst[self.GridSlot] else 0
		if self.GridSize>0 then
			PlayerGUI.ScreenGui.Snap.Text="Snap: "..tostring(self.GridLst[self.GridSlot]).." stud"
		else
			PlayerGUI.ScreenGui.Snap.Text="Snap: "..'0'.." stud"
		end
		
	end
end


function ClientPlacer:CycleGrid(_,state,obj)
	if state==Enum.UserInputState.Begin then
		if self.Mode=="Delete" then return end
		if self.GridSize>0 then
			self.GridSlot+=1
			if self.GridSlot==#self.GridLst+1 then
				self.GridSlot=1
			end
			if self.GridSize>0 then
				self.GridSize=self.GridLst[self.GridSlot]
			end
			PlayerGUI.ScreenGui.Snap.Text="Snap: "..tostring(self.GridLst[self.GridSlot]).." stud"
		end
	end
end

function ClientPlacer:Destroy()
	if self.Preview then
		self.Preview:Destroy()
		self.Preview=nil
	end
	self:EnableVisualBorder(false)
	self.Mode="Place"
	PlayerGUI.MobileDeleteUI.ImageLabel.Visible=false
	DeleterBoxOutline.Adornee=nil
	DeleterBoxOutline.Parent=nil
	PlayerGUI.ScreenGui.Enabled=false
	PlayerGUI.PlacementView.Main.Visible=false
	RunService:UnbindFromRenderStep(PREVIEW_RENDER)
	ContextActionService:UnbindAction(PLACE_ACTION)
	ContextActionService:UnbindAction(ROTATE_ACTION)
	ContextActionService:UnbindAction(DELETE_ACTION)
	ContextActionService:UnbindAction(CYCLE_ACTION)
	ContextActionService:UnbindAction(SNAP_ACTION)
	ContextActionService:UnbindAction(CYCLE_GRID)
end

return ClientPlacer
