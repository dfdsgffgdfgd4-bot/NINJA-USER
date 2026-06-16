local Players = game:GetService("Players")
local LP = Players.LocalPlayer

local function createVisualTag(head)
    if head:FindFirstChild("NinjaTagBillboard") then return end
    local bb = Instance.new("BillboardGui", head)
    bb.Name = "NinjaTagBillboard"
    bb.Size = UDim2.new(0, 150, 0, 50) 
    bb.StudsOffset = Vector3.new(0, 3, 0)
    bb.AlwaysOnTop = true
    bb.LightInfluence = 0
    local tl = Instance.new("TextLabel", bb)
    tl.Size = UDim2.new(1, 0, 1, 0)
    tl.BackgroundTransparency = 1
    tl.Text = "NINJA USER"
    tl.TextColor3 = Color3.fromRGB(255, 0, 0)
    tl.Font = Enum.Font.GothamBold
    tl.TextScaled = true
    tl.TextStrokeTransparency = 0
    tl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
end

local function markMe()
    local char = LP.Character or LP.CharacterAdded:Wait()
    local head = char:WaitForChild("Head", 10)
    if head then
        if not head:FindFirstChild("NinjaID") then
            Instance.new("StringValue", head).Name = "NinjaID"
        end
        createVisualTag(head)
    end
end

task.spawn(markMe)
LP.CharacterAdded:Connect(markMe)

task.spawn(function()
    while task.wait(2) do
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local h = p.Character:FindFirstChild("Head")
                if h and h:FindFirstChild("NinjaID") then
                    createVisualTag(h)
                end
            end
        end
    end
end)
