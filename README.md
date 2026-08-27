#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <functional>
#include <cstdlib>
#include <cpr/cpr.h>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

json mock_database = {
    {"user_123", {
        {"name", "陳小明"},
        {"accounts", {{"savings", 150000}}},
        {"cards", {
            {"card_8888", {{"status", "active"}, {"type", "無限卡"}}}
        }}
    }}
};

std::string get_account_balance(const json& args) {
    std::string user_id = args.value("user_id", "");
    std::string account_type = args.value("account_type", "savings");

    if (!mock_database.contains(user_id)) {
        return json({{"error", "找不到使用者"}}).dump();
    }
    
    int balance = mock_database[user_id]["accounts"].value(account_type, 0);
    return json({
        {"user_id", user_id},
        {"account_type", account_type},
        {"balance", balance}
    }).dump();
}

std::string block_credit_card(const json& args) {
    std::string user_id = args.value("user_id", "");
    std::string card_number = args.value("card_number", "");
    std::string reason = args.value("reason", "");

    if (!mock_database.contains(user_id) || !mock_database[user_id]["cards"].contains(card_number)) {
        return json({{"status", "failed"}, {"message", "卡號無效或不符合"}}).dump();
    }

    mock_database[user_id]["cards"][card_number]["status"] = "blocked";
    return json({
        {"status", "success"},
        {"message", "卡片 " + card_number + " 已成功掛失並凍結。原因：" + reason},
        {"reference_id", "TXN-9982341"}
    }).dump();
}

class BankingAgent {
private:
    std::string api_key;
    std::string api_url;
    json contents;
    std::unordered_map<std::string, std::function<std::string(const json&)>> tool_map;

    json get_tools_schema() {
        return json::array({
            {
                {"function_declarations", json::array({
                    {
                        {"name", "get_account_balance"},
                        {"description", "查詢使用者帳戶餘額"},
                        {"parameters", {
                            {"type", "OBJECT"},
                            {"properties", {
                                {"user_id", {{"type", "STRING"}, {"description", "使用者ID"}}},
                                {"account_type", {{"type", "STRING"}, {"description", "帳戶類型，預設 savings"}}}
                            }},
                            {"required", json::array({"user_id"})}
                        }}
                    },
                    {
                        {"name", "block_credit_card"},
                        {"description", "自動辦理信用卡掛失與凍結"},
                        {"parameters", {
                            {"type", "OBJECT"},
                            {"properties", {
                                {"user_id", {{"type", "STRING"}, {"description", "使用者ID"}}},
                                {"card_number", {{"type", "STRING"}, {"description", "信用卡號"}}},
                                {"reason", {{"type", "STRING"}, {"description", "掛失原因"}}}
                            }},
                            {"required", json::array({"user_id", "card_number", "reason"})}
                        }}
                    }
                })}
            }
        });
    }

public:
    BankingAgent() {
        const char* env_p = std::getenv("GEMINI_API_KEY");
        if (!env_p) {
            std::cerr << "錯誤：請設定 GEMINI_API_KEY 環境變數！" << std::endl;
            std::exit(1);
        }
        api_key = std::string(env_p);
        api_url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=" + api_key;

        tool_map["get_account_balance"] = get_account_balance;
        tool_map["block_credit_card"] = block_credit_card;

        contents = json::array();
    }

    void send_message(const std::string& user_input) {
        std::cout << "\n👤 使用者: " << user_input << std::endl;

        contents.push_back({
            {"role", "user"},
            {"parts", json::array({{{"text", user_input}}})}
        });

        bool loop = true;
        while (loop) {
            json request_body = {
                {"contents", contents},
                {"tools", get_tools_schema()},
                {"system_instruction", {
                    {"parts", json::array({{{"text", 
                        "你是一家數位銀行的 AI 客服 Agent，名字叫「FutureBank 智能專員」。\n"
                        "注意事項：\n"
                        "1. 語氣禮貌、專業。\n"
                        "2. 嚴格依據系統回傳的 Tool 結果回答，切勿虛構金融資料。\n"
                        "3. 當前對話預設的用戶身分 ID 為 'user_123'。"
                    }}})}
                }}
            };

            auto response = cpr::Post(
                cpr::Url{api_url},
                cpr::Header{{"Content-Type", "application/json"}},
                cpr::Body{request_body.dump()}
            );

            if (response.status_code != 200) {
                std::cerr << "API 請求失敗，狀態碼: " << response.status_code << std::endl;
                std::cerr << response.text << std::endl;
                return;
            }

            json res_json = json::parse(response.text);
            json candidate_content = res_json["candidates"][0]["content"];
            
            contents.push_back(candidate_content);

            bool has_function_call = false;
            for (const auto& part : candidate_content["parts"]) {
                if (part.contains("functionCall")) {
                    has_function_call = true;
                    std::string func_name = part["functionCall"]["name"];
                    json func_args = part["functionCall"]["args"];

                    std::string tool_result_str = "";
                    if (tool_map.count(func_name)) {
                        tool_result_str = tool_map[func_name](func_args);
                    }

                    json function_response_part = {
                        {"functionResponse", {
                            {"name", func_name},
                            {"response", {{"result", json::parse(tool_result_str)}}}
                        }}
                    };

                    contents.push_back({
                        {"role", "user"},
                        {"parts", json::array({function_response_part})}
                    });
                }
            }

            if (!has_function_call) {
                for (const auto& part : candidate_content["parts"]) {
                    if (part.contains("text")) {
                        std::cout << "🤖 AI 客服: " << part["text"].get<std::string>() << std::endl;
                    }
                }
                loop = false;
            }
        }
    }
};

int main() {
    BankingAgent agent;

    agent.send_message("嗨，我想看一下我活存帳戶還有多少錢？");
    agent.send_message("糟糕！我的信用卡 card_8888 好像剛才掉在計程車上了，請幫我掛失！");

    std::cout << "\n[後端數據庫檢查] 當前卡片狀態: " 
              << mock_database["user_123"]["cards"]["card_8888"].dump() 
              << std::endl;

    return 0;
}
