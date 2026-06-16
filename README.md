--[[ 
    NINJA USER - VERSÃO DELTA (SIMPLIFICADA)
    Só aparece para quem também estiver usando este script.
]]

local Players = game:GetService("Players")
local LP = Players.LocalPlayer

-- Função para criar a tag visual
local function createVisualTag(head)
    if head:FindFirstChild("NinjaTagBillboard") then return end

    local billboardGui = Instance.new("BillboardGui")
    billboardGui.Name = "NinjaTagBillboard"
    billboardGui.Size = UDim2.new(0, 150, 0, 50) 
    billboardGui.StudsOffset = Vector3.new(0, 3, 0)
    billboardGui.AlwaysOnTop = true
    billboardGui.LightInfluence = 0
    billboardGui.Parent = head

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = "NINJA USER"
    textLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
    textLabel.Font = Enum.Font.GothamBold
    textLabel.TextScaled = true
    textLabel.TextStrokeTransparency = 0
    textLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    textLabel.Parent = billboardGui
end

-- 1. MARCAR VOCÊ COMO NINJA (Para que outros te vejam)
-- Criamos um objeto invisível chamado "NinjaID" na sua cabeça
local function markMe()
    local char = LP.Character or LP.CharacterAdded:Wait()
    local head = char:WaitForChild("Head", 10)
    if head then
        if not head:FindFirstChild("NinjaID") then
            local id = Instance.new("StringValue")
            id.Name = "NinjaID"
            id.Parent = head
        end
        createVisualTag(head) -- Você também vê a sua tag
    end
end

task.spawn(markMe)
LP.CharacterAdded:Connect(markMe)

-- 2. LOOP DE DETECÇÃO (Para você ver os outros Ninjas)
-- O script fica olhando se os outros jogadores têm o "NinjaID" na cabeça
task.spawn(function()
    while task.wait(2) do -- Verifica a cada 2 segundos para não dar lag
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LP and player.Character then
                local head = player.Character:FindFirstChild("Head")
                if head and head:FindFirstChild("NinjaID") then
                    -- Se ele tem o NinjaID, significa que ele também rodou o script!
                    createVisualTag(head)
                end
            end
        end
    end
end)

print("Ninja User Delta Ativado!")
