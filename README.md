-- LocalScript for Delta Executor (Single Remote Lock)

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

-- Network Prefix to identify chat messages
local CHAT_PREFIX = "[DELTA_CHAT_v3]:"

-- 1. Find and lock onto a single target RemoteEvent
local SelectedRemote = nil

for _, obj in pairs(game:GetDescendants()) do
	if obj:IsA("RemoteEvent") then
		SelectedRemote = obj
		break -- Stop searching immediately after finding the first one
	end
end

-- UI Setup
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "RobloxChatSimulator"
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 100 
screenGui.Parent = PlayerGui

-- Circular Chat Icon Button
local toggleButton = Instance.new("ImageButton")
toggleButton.Name = "ToggleChatButton"
toggleButton.Size = UDim2.new(0, 36, 0, 36)
toggleButton.Position = UDim2.new(0, 10, 0, 10)
toggleButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
toggleButton.BackgroundTransparency = 0.3
toggleButton.BorderSizePixel = 0
toggleButton.Image = "rbxassetid://116758803583558" 
toggleButton.ImageColor3 = Color3.fromRGB(255, 255, 255)
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

-- Solid Black Chat Window
local chatFrame = Instance.new("Frame")
chatFrame.Name = "ChatFrame"
chatFrame.Size = UDim2.new(0, 270, 0, 140)
chatFrame.Position = UDim2.new(0, 10, 0, 52)
chatFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
chatFrame.BackgroundTransparency = 0.25
chatFrame.BorderSizePixel = 0
chatFrame.Visible = true
chatFrame.Parent = screenGui

local chatCorner = Instance.new("UICorner")
chatCorner.CornerRadius = UDim.new(0, 8)
chatCorner.Parent = chatFrame

-- Scrolling Frame
local scrollFrame = Instance.new("ScrollingFrame")
scrollFrame.Name = "MessageScroll"
scrollFrame.Size = UDim2.new(1, -8, 1, -38)
scrollFrame.Position = UDim2.new(0, 4, 0, 4)
scrollFrame.BackgroundTransparency = 1
scrollFrame.BorderSizePixel = 0
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.ScrollBarThickness = 3
scrollFrame.Parent = chatFrame

local uiList = Instance.new("UIListLayout")
uiList.SortOrder = Enum.SortOrder.LayoutOrder
uiList.Padding = UDim.new(0, 2)
uiList.Parent = scrollFrame

-- Text Input Box
local chatBox = Instance.new("TextBox")
chatBox.Name = "ChatBox"
chatBox.Size = UDim2.new(1, -8, 0, 26)
chatBox.Position = UDim2.new(0, 4, 1, -30)
chatBox.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
chatBox.BackgroundTransparency = 0.2
chatBox.BorderSizePixel = 0
chatBox.ClearTextOnFocus = false
chatBox.Font = Enum.Font.SourceSans
chatBox.PlaceholderText = "To chat click here..."
chatBox.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
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

-- Dragging Functionality
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

-- UI Display Function
local function addMessage(speaker, message, color)
	local textLabel = Instance.new("TextLabel")
	textLabel.Size = UDim2.new(1, 0, 0, 0)
	textLabel.AutomaticSize = Enum.AutomaticSize.Y
	textLabel.BackgroundTransparency = 1
	textLabel.Font = Enum.Font.SourceSansBold
	textLabel.TextSize = 13
	textLabel.TextXAlignment = Enum.TextXAlignment.Left
	textLabel.TextWrapped = true
	textLabel.RichText = true
	
	if speaker then
		textLabel.Text = string.format("<font color=\"rgb(200,200,200)\">[%s]:</font> <font color=\"rgb(%d,%d,%d)\">%s</font>", 
			speaker, color.R * 255, color.G * 255, color.B * 255, message)
	else
		textLabel.Text = string.format("<font color=\"rgb(255,100,100)\">%s</font>", message)
	end
	
	textLabel.Parent = scrollFrame
	scrollFrame.CanvasPosition = Vector2.new(0, scrollFrame.AbsoluteCanvasSize.Y)
end

-- Helper to parse incoming chat payloads
local function parseIncoming(data)
	if type(data) == "string" and string.sub(data, 1, #CHAT_PREFIX) == CHAT_PREFIX then
		local payload = string.sub(data, #CHAT_PREFIX + 1)
		local splitPos = string.find(payload, ":")
		
		if splitPos then
			local sender = string.sub(payload, 1, splitPos - 1)
			local message = string.sub(payload, splitPos + 1)
			
			if sender ~= Player.Name then
				addMessage(sender, message, Color3.fromRGB(100, 200, 255))
			end
		end
	end
end

-- Listen ONLY to the selected remote
if SelectedRemote then
	SelectedRemote.OnClientEvent:Connect(function(...)
		local args = {...}
		for _, v in pairs(args) do
			parseIncoming(v)
		end
	end)
end

-- Send message via ONLY the selected remote
chatBox.FocusLost:Connect(function(enterPressed)
	if enterPressed and chatBox.Text ~= "" then
		local msg = chatBox.Text
		chatBox.Text = ""
		
		-- Display locally
		addMessage(Player.Name, msg, Color3.fromRGB(255, 255, 255))
		
		if SelectedRemote then
			local encodedMessage = CHAT_PREFIX .. Player.Name .. ":" .. msg
			pcall(function()
				SelectedRemote:FireServer(encodedMessage)
			end)
		else
			addMessage(nil, "Error: No valid RemoteEvent found in this game.", Color3.fromRGB(255, 100, 100))
		end
	end
end)

-- Initial Welcome Message
task.wait(0.5)
if SelectedRemote then
	addMessage(nil, "Locked onto remote: " .. SelectedRemote.Name, Color3.fromRGB(255, 255, 255))
else
	addMessage(nil, "No RemoteEvent found in game.", Color3.fromRGB(255, 100, 100))
end
