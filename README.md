-- LocalScript for Delta Executor (HTTP Global Chat + Real Chat Auto-Relay)

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TextChatService = game:GetService("TextChatService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

-- Detector for Executor HTTP Request Function
local requestFunc = (syn and syn.request) or (http and http.request) or request or http_request
if not requestFunc then
	warn("Your executor does not support HTTP requests!")
	return
end

---------------------------------------------------------
-- CONFIGURATION
---------------------------------------------------------
local BIN_ID = "6a6ce0dcf5f4af5e29db0374"
local MASTER_KEY = "$2a$10$bRRpF3qqRayEUQNrEW4z8uOFuHeBuAF.2NI.KJ/Us9Mcldjf2cHCa"
local BIN_URL = "https://api.jsonbin.io/v3/b/" .. BIN_ID
---------------------------------------------------------

local currentTab = "Global"

-- UI Setup
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "DeltaHttpChat"
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 100
screenGui.Parent = PlayerGui

-- Circular Toggle Button
local toggleButton = Instance.new("ImageButton")
toggleButton.Name = "ToggleChatButton"
toggleButton.Size = UDim2.new(0, 36, 0, 36)
toggleButton.Position = UDim2.new(0, 10, 0, 10)
toggleButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
toggleButton.BackgroundTransparency = 0.3
toggleButton.BorderSizePixel = 0
toggleButton.Image = "rbxassetid://116758803583558"
toggleButton.Parent = screenGui

local buttonCorner = Instance.new("UICorner")
buttonCorner.CornerRadius = UDim.new(1, 0)
buttonCorner.Parent = toggleButton

local buttonPadding = Instance.new("UIPadding")
buttonPadding.PaddingTop = UDim.new(0, 6)
buttonPadding.PaddingBottom = UDim.new(0, 6)
buttonPadding.PaddingLeft = UDim.new(0, 6)
buttonPadding.PaddingRight = UDim.new(0, 6)
buttonPadding.Parent = toggleButton

-- Main Chat Frame
local chatFrame = Instance.new("Frame")
chatFrame.Name = "ChatFrame"
chatFrame.Size = UDim2.new(0, 270, 0, 160)
chatFrame.Position = UDim2.new(0, 10, 0, 52)
chatFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
chatFrame.BackgroundTransparency = 0.25
chatFrame.BorderSizePixel = 0
chatFrame.Parent = screenGui

local chatCorner = Instance.new("UICorner")
chatCorner.CornerRadius = UDim.new(0, 8)
chatCorner.Parent = chatFrame

-- Tab Bar
local tabBar = Instance.new("Frame")
tabBar.Name = "TabBar"
tabBar.Size = UDim2.new(1, -8, 0, 20)
tabBar.Position = UDim2.new(0, 4, 0, 4)
tabBar.BackgroundTransparency = 1
tabBar.Parent = chatFrame

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0, 4)
tabLayout.Parent = tabBar

local function createTabBtn(text)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0.5, -2, 1, 0)
	btn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	btn.BackgroundTransparency = 0.2
	btn.BorderSizePixel = 0
	btn.Font = Enum.Font.SourceSansBold
	btn.Text = text
	btn.TextColor3 = Color3.fromRGB(180, 180, 180)
	btn.TextSize = 12
	btn.Parent = tabBar

	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, 4)
	c.Parent = btn

	return btn
end

local globalBtn = createTabBtn("Global Chat")
local realChatBtn = createTabBtn("Real Chat")

-- Scrolling Message Container
local scrollFrame = Instance.new("ScrollingFrame")
scrollFrame.Size = UDim2.new(1, -8, 1, -62)
scrollFrame.Position = UDim2.new(0, 4, 0, 28)
scrollFrame.BackgroundTransparency = 1
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.ScrollBarThickness = 3
scrollFrame.Parent = chatFrame

local uiList = Instance.new("UIListLayout")
uiList.SortOrder = Enum.SortOrder.LayoutOrder
uiList.Padding = UDim.new(0, 2)
uiList.Parent = scrollFrame

-- Input Box
local chatBox = Instance.new("TextBox")
chatBox.Size = UDim2.new(1, -8, 0, 26)
chatBox.Position = UDim2.new(0, 4, 1, -30)
chatBox.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
chatBox.BackgroundTransparency = 0.2
chatBox.PlaceholderText = "Type message here..."
chatBox.Text = ""
chatBox.TextColor3 = Color3.fromRGB(255, 255, 255)
chatBox.TextSize = 13
chatBox.TextXAlignment = Enum.TextXAlignment.Left
chatBox.Parent = chatFrame

local boxCorner = Instance.new("UICorner")
boxCorner.CornerRadius = UDim.new(0, 4)
boxCorner.Parent = chatBox

local boxPadding = Instance.new("UIPadding")
boxPadding.PaddingLeft = UDim.new(0, 6)
boxPadding.Parent = chatBox

-- Dragging Helper
local function makeDraggable(guiObject)
	local dragging = false
	local dragInput, dragStart, startPos
	local hasMoved = false

	local function update(input)
		local delta = input.Position - dragStart
		if delta.Magnitude > 5 then
			hasMoved = true
		end
		guiObject.Position = UDim2.new(
			startPos.X.Scale, 
			startPos.X.Offset + delta.X, 
			startPos.Y.Scale, 
			startPos.Y.Offset + delta.Y
		)
	end

	guiObject.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			hasMoved = false
			dragStart = input.Position
			startPos = guiObject.Position

			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
				end
			end)
		end
	end)

	guiObject.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if input == dragInput and dragging then
			update(input)
		end
	end)

	return function()
		return hasMoved
	end
end

makeDraggable(chatFrame)
local buttonMoved = makeDraggable(toggleButton)

toggleButton.MouseButton1Click:Connect(function()
	if not buttonMoved() then
		chatFrame.Visible = not chatFrame.Visible
	end
end)

-- Tab Switcher Logic
local function switchTab(tab)
	currentTab = tab
	if currentTab == "Global" then
		globalBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
		realChatBtn.TextColor3 = Color3.fromRGB(150, 150, 150)
		chatBox.Visible = true
		scrollFrame.Size = UDim2.new(1, -8, 1, -62)
	else
		realChatBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
		globalBtn.TextColor3 = Color3.fromRGB(150, 150, 150)
		chatBox.Visible = false
		scrollFrame.Size = UDim2.new(1, -8, 1, -34)
	end

	for _, child in ipairs(scrollFrame:GetChildren()) do
		if child:IsA("TextLabel") then
			local category = child:GetAttribute("Category")
			if category == "System" or category == currentTab then
				child.Visible = true
			else
				child.Visible = false
			end
		end
	end
	scrollFrame.CanvasPosition = Vector2.new(0, scrollFrame.AbsoluteCanvasSize.Y)
end

globalBtn.MouseButton1Click:Connect(function() switchTab("Global") end)
realChatBtn.MouseButton1Click:Connect(function() switchTab("RealChat") end)

-- UI Display Function
local function addMessage(speaker, message, category, color)
	category = category or "Global"
	
	local textLabel = Instance.new("TextLabel")
	textLabel.Size = UDim2.new(1, 0, 0, 0)
	textLabel.AutomaticSize = Enum.AutomaticSize.Y
	textLabel.BackgroundTransparency = 1
	textLabel.Font = Enum.Font.SourceSansBold
	textLabel.TextSize = 13
	textLabel.TextXAlignment = Enum.TextXAlignment.Left
	textLabel.TextWrapped = true
	textLabel.RichText = true
	textLabel:SetAttribute("Category", category)

	if speaker then
		if category == "RealChat" then
			textLabel.Text = string.format("<font color=\"rgb(255,220,100)\">[%s]:</font> <font color=\"rgb(255,255,255)\">%s</font>", 
				speaker, message)
		else
			local c = color or Color3.fromRGB(100, 200, 255)
			textLabel.Text = string.format("<font color=\"rgb(200,200,200)\">[%s]:</font> <font color=\"rgb(%d,%d,%d)\">%s</font>", 
				speaker, c.R * 255, c.G * 255, c.B * 255, message)
		end
	else
		local c = color or Color3.fromRGB(255, 100, 100)
		textLabel.Text = string.format("<font color=\"rgb(%d,%d,%d)\">%s</font>", c.R * 255, c.G * 255, c.B * 255, message)
	end

	textLabel.Parent = scrollFrame
	
	if category == "System" or category == currentTab then
		textLabel.Visible = true
	else
		textLabel.Visible = false
	end
	
	scrollFrame.CanvasPosition = Vector2.new(0, scrollFrame.AbsoluteCanvasSize.Y)
end

---------------------------------------------------------
-- HTTP PUSH (BROADCAST HELPER)
---------------------------------------------------------
local function broadcastToJSON(senderName, messageText)
	task.spawn(function()
		pcall(function()
			local fetchResponse = requestFunc({
				Url = BIN_URL .. "/latest",
				Method = "GET",
				Headers = { ["X-Master-Key"] = MASTER_KEY }
			})

			local currentList = {}
			if fetchResponse and fetchResponse.Body then
				local decoded = HttpService:JSONDecode(fetchResponse.Body)
				currentList = decoded.record or {}
			end

			local newEntry = {
				user = senderName,
				msg = messageText,
				time = os.time()
			}
			table.insert(currentList, newEntry)

			if #currentList > 20 then
				table.remove(currentList, 1)
			end

			requestFunc({
				Url = BIN_URL,
				Method = "PUT",
				Headers = {
					["Content-Type"] = "application/json",
					["X-Master-Key"] = MASTER_KEY
				},
				Body = HttpService:JSONEncode(currentList)
			})
		end)
	end)
end

---------------------------------------------------------
-- REAL IN-GAME CHAT LISTENER & RELAY TO JSONBIN
---------------------------------------------------------
local function handleIncomingRealChat(senderName, messageText)
	-- Shows in your Real Chat tab locally
	addMessage(senderName, messageText, "RealChat")
	-- Relays over JSONBin so your friend sees it in their Global tab
	broadcastToJSON("[RC] " .. senderName, messageText)
end

-- Hook into modern TextChatService
if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
	local textChannels = TextChatService:FindFirstChild("TextChannels")
	if textChannels then
		for _, channel in ipairs(textChannels:GetChildren()) do
			if channel:IsA("TextChannel") then
				channel.MessageReceived:Connect(function(msg)
					if msg.TextSource then
						local senderPlayer = Players:GetPlayerByUserId(msg.TextSource.UserId)
						local name = senderPlayer and senderPlayer.Name or "Unknown"
						handleIncomingRealChat(name, msg.Text)
					end
				end)
			end
		end
	end
end

-- Fallback for Legacy Chat engine
local chatEvents = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
if chatEvents then
	local onMessageDone = chatEvents:FindFirstChild("OnMessageDoneFiltering")
	if onMessageDone then
		onMessageDone.OnClientEvent:Connect(function(data)
			if data and data.FromSpeaker and data.Message then
				handleIncomingRealChat(data.FromSpeaker, data.Message)
			end
		end)
	end
end

---------------------------------------------------------
-- NETWORK SYNC (POLL & BROADCAST)
---------------------------------------------------------

local processedMessages = {}

-- Poll for new messages every 2 seconds
task.spawn(function()
	while task.wait(2) do
		pcall(function()
			local response = requestFunc({
				Url = BIN_URL .. "/latest",
				Method = "GET",
				Headers = {
					["X-Master-Key"] = MASTER_KEY
				}
			})

			if response and response.Body then
				local decoded = HttpService:JSONDecode(response.Body)
				local chatList = decoded.record

				if type(chatList) == "table" then
					for _, item in ipairs(chatList) do
						local msgKey = item.user .. ":" .. item.msg .. ":" .. tostring(item.time)
						if not processedMessages[msgKey] then
							processedMessages[msgKey] = true
							
							if item.user ~= Player.Name then
								addMessage(item.user, item.msg, "Global", Color3.fromRGB(100, 200, 255))
							end
						end
					end
				end
			end
		end)
	end
end)

-- Send new message when pressing Enter
chatBox.FocusLost:Connect(function(enterPressed)
	if enterPressed and chatBox.Text ~= "" then
		local msg = chatBox.Text
		chatBox.Text = ""

		addMessage(Player.Name, msg, "Global", Color3.fromRGB(255, 255, 255))
		broadcastToJSON(Player.Name, msg)
	end
end)

addMessage(nil, "Connected to Global HTTP Chat!", "System", Color3.fromRGB(100, 255, 100))
switchTab("Global")
