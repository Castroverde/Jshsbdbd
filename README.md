local VERSION = "2.4.6"
local letters = {"A","B","C","D","E","F","G","H","I","J","K","L","M","N","O","P","Q","R","S","T","U","V","W","X","Y","Z"}
local userCooldowns = {}
local currentQuestion
local questionAnsweredBy
local quizRunning = false
local blockedPlayers = {}
local whiteListedPlayers = {}
local mode = "Quiz"
local answeredCorrectly = {}
local submittedAnswer = {}
local awaitingAnswer = false
local questionPoints = 1
local timeSinceLastMessage = tick()
local placeId = game.PlaceId
local quizCooldown = false
local answerOptionsSaid = 0
local minMessageCooldown = 2.3
local whiteListEnabled = false

local settings = {
    questionTimeout = 10,
    userCooldown = 5,
    sendLeaderBoardAfterQuestions = 0, -- only send quiz LB at end of quiz by default
    automaticLeaderboards = true,
    automaticCurrentQuizLeaderboard = false,
    automaticServerQuizLeaderboard = true,
    signStatus = true,
    romanNumbers = true,
    autoplay = false,
    repeatTagged = true,
    sendDetailedCategorylist = false, -- off by default since sending it this way tends to trigger the filter. Blame Roblox
    removeLeavingPlayersFromLB = true,
}

local numberMap = {
    {1000, 'M'}, {900, 'CM'}, {500, 'D'}, {400, 'CD'}, {100, 'C'},
    {90, 'XC'}, {50, 'L'}, {40, 'XL'}, {10, 'X'}, {9, 'IX'},
    {5, 'V'}, {4, 'IV'}, {1, 'I'}
}

-- Converts integer to Roman numeral
local function intToRoman(num)
    local roman = ""
    while num > 0 do
        for _, v in pairs(numberMap) do
            local romanChar = v[2]
            local int = v[1]
            while num >= int do
                roman = roman..romanChar
                num = num - int
            end
        end
    end
    return roman
end

local oldChat = textChatService.ChatVersion ~= Enum.ChatVersion.TextChatService

-- Send chat message
local function Chat(msg)
    if not oldChat then
        textChatService.TextChannels.RBXGeneral:SendAsync(msg)
    else
        replicatedStorage.DefaultChatSystemChatEvents.SayMessageRequest:FireServer(msg, "All")
    end
end

-- Shuffle table
local function Shuffle(tbl)
    local rng = Random.new()
    for i = #tbl, 2, -1 do
        local j = rng:NextInteger(1, i)
        tbl[i], tbl[j] = tbl[j], tbl[i]
    end
    return tbl
end

-- Round number to specified decimal places
local function roundNumber(num, numDecimalPlaces)
    return tonumber(string.format("%." .. (numDecimalPlaces or 0) .. "f", num))
end

-- Notify user with a message
local notifyBindable = Instance.new("BindableFunction")
local function notify(title, text, configTable, duration)
    configTable = configTable or {}
    StarterGui:SetCore("SendNotification", {
        Title = title,
        Text = text,
        Callback = notifyBindable,
        Button1 = configTable.Button1,
        Button2 = configTable.Button2,
        Duration = duration
    })
    if configTable.Callback then
        notifyBindable.OnInvoke = configTable.Callback
    end
end

-- Set global environment variable
local function setgenv(key, value)
    pcall(function()
        getgenv()[key] = value
    end)
end

if getgenv and getgenv().QUIZBOT_RUNNING then
    notify("quizbot is already running", "You cannot run two instances of quizbot at once")
    return
end
setgenv("QUIZBOT_RUNNING", true)

-- Copy latest script to clipboard
local function copyLatestScript()
    setclipboard('loadstring(game:HttpGet("https://raw.githubusercontent.com/Damian-11/quizbot/master/quizbot.luau"))()')
    notify("Script copied", "The latest version of quizbot has been copied to your clipboard")
end

local DATA_FILENAME = "quizbot_data.json"
local data = {}

if isfile and isfile(DATA_FILENAME) then
    data = HttpService:JSONDecode(readfile(DATA_FILENAME))
end

-- Write data to file
local function writeToDataFile(key, value)
    if not writefile then
        notify("Can't write to file", "Your exploit does not support the writefile function. Your settings will not be saved.")
        return
    end
    data[key] = value
    writefile(DATA_FILENAME, HttpService:JSONEncode(data))
end

local function getDataFileValue(key)
    return data[key]
end

-- Check for latest version
local LATEST_VERSION_URL = "https://raw.githubusercontent.com/Damian-11/quizbot/master/version_number.luau"
local success, LATEST_VERSION = pcall(function()
    return loadstring(game:HttpGet(LATEST_VERSION_URL))()
end)

local outdated = success and LATEST_VERSION and VERSION ~= LATEST_VERSION
if outdated and not getDataFileValue("disableVersionAlert") then
    notify(
        "Outdated quizbot version",
        "You are running an outdated version of quizbot. Click the button below to copy the latest version of this script",
        { Callback = copyLatestScript, Button1 = "Copy latest version" },
        10
    )
end

-- Escape special characters in string
local function EscapePattern(pattern)
    local escapePattern = "[%(%)%.%%%+%-%*%?%[%]%^%$]"
    return string.gsub(pattern, escapePattern, "%%%1")
end

local antiFilteringDone
local importantMessageSent
local messageBeforeFilter
local answeredByAltMessage
local mainQuestionSent
local messageFiltered
local lastMessageTime

-- Send message with rate limit management
local function SendMessageWhenReady(message, important, altMessage)
    if not quizRunning then return end
    if not settings.repeatTagged then important = false end
    if important then
        importantMessageSent = true
        messageBeforeFilter = message
        answeredByAltMessage = altMessage
        messageFiltered = false
        antiFilteringDone = false
    end
    if tick() - timeSinceLastMessage >= minMessageCooldown then
        Chat(message)
        timeSinceLastMessage = tick()
    else
        task.wait(minMessageCooldown - (tick() - timeSinceLastMessage))
        if not quizRunning then return end
        Chat(message)
        timeSinceLastMessage = tick()
    end
    if important then
        lastMessageTime = tick()
        while (not antiFilteringDone or mainQuestionSent) and quizRunning do
            task.wait()
            if tick() - lastMessageTime > 4.5 then
                notify("Error sending quiz message", "Chat rate limit reached. Avoid sending messages in the chat while a quiz is running")
                messageFiltered = true
                break
            end
        end
    end
    importantMessageSent = false
end

-- Function to update sign text
local function UpdateSignText(text)
    if not boothGame or not settings.signStatus or not text then return end
    local sign = localPlayer.Character:FindFirstChild("Text Sign") or localPlayer.Backpack:FindFirstChild("Text Sign")
    if not sign then return end
    signRemote:FireServer("SignServer")
    changeSignTextRemote:FireServer(text)
end

-- Calculate read time for a text message
local function CalculateReadTime(text)
    local timeToWait = #string.split(text, " ") * 0.4
    if timeToWait < minMessageCooldown then
        timeToWait = minMessageCooldown
    end
    return timeToWait
end

--- Question OOP ---
local question = {}
question.__index = question

function question.New(quesitonText, options, value)
    local newQuestion = {}
    newQuestion.mainQuestion = quesitonText
    newQuestion.answers = options
    newQuestion.rightAnswer = letters[1]
    newQuestion.rightAnswerIndex = 1
    value = value or 1
    newQuestion.value = value
    setmetatable(newQuestion, question)
    return newQuestion
end

function question:Ask()
    if not quizRunning then
        return false
    end
    answerOptionsSaid = 0
    local rightAnswerBeforeShuffle = self.answers[self.rightAnswerIndex]
    self.answers = Shuffle(self.answers)
    self.rightAnswerIndex = table.find(self.answers, rightAnswerBeforeShuffle)
    self.rightAnswer = letters[self.rightAnswerIndex]
    if self.value > 1 then
        SendMessageWhenReady("⭐ | "..self.value.."x points for question")
        task.wait(2)
    end
    questionAnsweredBy = nil
    UpdateSignText(self.mainQuestion)
    currentQuestion = self
    questionPoints = self.value
    mainQuestionSent = true
    SendMessageWhenReady("🎙️ | "..self.mainQuestion, true)
    if messageFiltered then
        task.wait(3)
        Chat("➡️ | Repeated filtering detected. Skipping to the next question...")
        return false
    end
    local waitTime = CalculateReadTime(self.mainQuestion)
    local timeWaited = 0
    while waitTime > timeWaited do
        if questionAnsweredBy or not quizRunning then
            return true
        end
        task.wait(0.1)
        timeWaited += 0.1
    end
    for i, v in ipairs(self.answers) do
        if questionAnsweredBy or not quizRunning then
            return true
        end
        if i ~= 1 then
            task.wait(CalculateReadTime(v))
        end
        if questionAnsweredBy or not quizRunning then
            return true
        end
        SendMessageWhenReady(letters[i]..")"..v, true)
        answerOptionsSaid = i
    end
end

--- Enhanced Categories ---
local categoryManager = {}
local categories = {}
categories.categoryList = {}
categories.numberOfDifficulties = {}
categoryManager.__index = categoryManager

local difficultyOrder = {"", "easy", "medium", "hard"}

function categoryManager.New(categoryName, difficulty)
    difficulty = difficulty or ""
    difficulty = string.lower(difficulty)
    if not categories.categoryList[categoryName] then
        categories.categoryList[categoryName] = {}
        categories.numberOfDifficulties[categoryName] = 0
    end
    if not categories.categoryList[categoryName][difficulty] then
        categories.categoryList[categoryName][difficulty] = {}
        categories.numberOfDifficulties[categoryName] += 1
        if not table.find(difficultyOrder, difficulty) then
            table.insert(difficultyOrder, difficulty)
        end
    end
    table.insert(categories.categoryList[categoryName][difficulty], {})
    local newCategory = categories.categoryList[categoryName][difficulty][#categories.categoryList[categoryName][difficulty]]
    setmetatable(newCategory, categoryManager)
    return newCategory
end

function categoryManager:Add(questionText, options, value, customQuestion)
    self = customQuestion or self
    local newQuestion = question.New(questionText, options, value)
    table.insert(self, newQuestion)
end

-- Enhanced Categories and Questions
local generalKnowledge = categoryManager.New("General Knowledge", "easy")
generalKnowledge:Add("What is the capital of France?", {"Paris", "London", "Berlin", "Madrid"})
generalKnowledge:Add("What is the largest planet in our solar system?", {"Jupiter", "Earth", "Mars", "Venus"})
generalKnowledge:Add("What year did the Titanic sink?", {"1912", "1905", "1920", "1898"})
generalKnowledge:Add("Who wrote 'To Kill a Mockingbird'?", {"Harper Lee", "Mark Twain", "Ernest Hemingway", "F. Scott Fitzgerald"})
generalKnowledge:Add("What is the chemical symbol for gold?", {"Au", "Ag", "Fe", "Pb"}, 2)

local history = categoryManager.New("History", "medium")
history:Add("Who was the first President of the United States?", {"George Washington", "Thomas Jefferson", "Abraham Lincoln", "John Adams"})
history:Add("In which year did World War II end?", {"1945", "1939", "1955", "1941"})
history:Add("What was the name of the ship that brought the Pilgrims to America in 1620?", {"Mayflower", "Santa Maria", "Endeavour", "Discovery"})
history:Add("Who was known as the 'Mad Monk' of Russia?", {"Rasputin", "Lenin", "Stalin", "Trotsky"})
history:Add("Which ancient civilization built the pyramids?", {"Egyptians", "Romans", "Greeks", "Mayans"}, 2)

local science = categoryManager.New("Science", "hard")
science:Add("What is the powerhouse of the cell?", {"Mitochondria", "Nucleus", "Ribosome", "Endoplasmic Reticulum"})
science:Add("What is the chemical formula for water?", {"H2O", "CO2", "NaCl", "O2"})
science:Add("Who developed the theory of relativity?", {"Albert Einstein", "Isaac Newton", "Galileo Galilei", "Nikola Tesla"})
science:Add("What is the hardest natural substance on Earth?", {"Diamond", "Gold", "Iron", "Quartz"})
science:Add("Who is known as the father of modern chemistry?", {"Antoine Lavoisier", "Dmitri Mendeleev", "Marie Curie", "Louis Pasteur"}, 2)

local geography = categoryManager.New("Geography", "medium")
geography:Add("Which country has the most natural lakes?", {"Canada", "Australia", "India", "Brazil"})
geography:Add("What is the longest river in the world?", {"Nile", "Amazon", "Yangtze", "Mississippi"})
geography:Add("What is the smallest country in the world?", {"Vatican City", "Monaco", "San Marino", "Liechtenstein"})
geography:Add("What is the capital of Australia?", {"Canberra", "Sydney", "Melbourne", "Perth"})
geography:Add("Which mountain is known as the tallest in the world?", {"Mount Everest", "K2", "Kangchenjunga", "Lhotse"}, 2)

local entertainment = categoryManager.New("Entertainment", "easy")
entertainment:Add("Who played Jack in the movie 'Titanic'?", {"Leonardo DiCaprio", "Brad Pitt", "Tom Cruise", "Johnny Depp"})
entertainment:Add("Which band released the album 'Abbey Road'?", {"The Beatles", "The Rolling Stones", "Pink Floyd", "Led Zeppelin"})
entertainment:Add("What is the name of Harry Potter's pet owl?", {"Hedwig", "Errol", "Pigwidgeon", "Crookshanks"})
entertainment:Add("Which TV show features a character named Sheldon Cooper?", {"The Big Bang Theory", "Friends", "How I Met Your Mother", "Modern Family"})
entertainment:Add("What is the highest-grossing film of all time?", {"Avatar", "Avengers: Endgame", "Titanic", "Star Wars: The Force Awakens"}, 2)

local literature = categoryManager.New("Literature", "hard")
literature:Add("Who wrote 'The Odyssey'?", {"Homer", "Virgil", "Ovid", "Sophocles"})
literature:Add("What is the pen name of Samuel Langhorne Clemens?", {"Mark Twain", "Lewis Carroll", "George Orwell", "H.P. Lovecraft"})
literature:Add("In which novel does the character 'Holden Caulfield' appear?", {"The Catcher in the Rye", "To Kill a Mockingbird", "1984", "Brave New World"})
literature:Add("Who is the author of the 'Harry Potter' series?", {"J.K. Rowling", "J.R.R. Tolkien", "C.S. Lewis", "Stephen King"})
literature:Add("What is the first book of the 'A Song of Ice and Fire' series?", {"A Game of Thrones", "A Clash of Kings", "A Storm of Swords", "A Feast for Crows"}, 2)

local sports = categoryManager.New("Sports", "medium")
sports:Add("Which country won the FIFA World Cup in 2018?", {"France", "Brazil", "Germany", "Argentina"})
sports:Add("Who holds the record for the most home runs in a single MLB season?", {"Barry Bonds", "Babe Ruth", "Hank Aaron", "Mark McGwire"})
sports:Add("What is the highest score possible in a single frame of bowling?", {"30", "20", "50", "40"})
sports:Add("Which sport is known as the 'king of sports'?", {"Soccer", "Basketball", "Tennis", "Cricket"})
sports:Add("Who is the all-time leading scorer in NBA history?", {"Kareem Abdul-Jabbar", "Michael Jordan", "LeBron James", "Kobe Bryant"}, 2)

local technology = categoryManager.New("Technology", "easy")
technology:Add("Who is the founder of Microsoft?", {"Bill Gates", "Steve Jobs", "Mark Zuckerberg", "Larry Page"})
technology:Add("What does 'HTML' stand for?", {"HyperText Markup Language", "HyperTool Markup Language", "HighText Markup Language", "HyperTech Markup Language"})
technology:Add("What year was the first iPhone released?", {"2007", "2005", "2003", "2009"})
technology:Add("Which company developed the Android operating system?", {"Google", "Apple", "Microsoft", "Samsung"})
technology:Add("What does 'CPU' stand for?", {"Central Processing Unit", "Computer Personal Unit", "Central Performance Unit", "Central Power Unit"}, 2)

-- Add more categories and questions as needed

--- Handle chat messages ---
local chatConnection
local joinConnection
local playerChatConnections = {}
if oldChat then
    settings.repeatTagged = false -- repeating tagged messages does not work on old chat system because of problems with different chat event (player.Chatted vs textChatService.MessageReceived)
    for _, player in players:GetPlayers() do
        if player ~= localPlayer then
            local connection = player.Chatted:Connect(function(message)
                startChatListening(message, player)
            end)