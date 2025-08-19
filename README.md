#include <stdio.h>
#include <string.h>
#include <ctype.h>

const char *sensitive_words[] = {
    "pwd", "psw", "password", "passwd", "auth", "cred", "token",
    "pri", "sharekey", "maskey", "md5key", "enkey", "userkey",
    "kmckey", "aes256", "privateSigned", "key", "keyconfirm",
    "keysecret", "keyaes"
};
const int sensitive_count = sizeof(sensitive_words) / sizeof(sensitive_words[0]);

void compute_prefix_function(const char *pattern, int *prefix) {
    int len = strlen(pattern);
    prefix[0] = 0;
    for (int i = 1; i < len; ++i) {
        int j = prefix[i - 1];
        while (j > 0 && pattern[i] != pattern[j]) {
            j = prefix[j - 1];
        }
        if (pattern[i] == pattern[j]) {
            j++;
        }
        prefix[i] = j;
    }
}

void kmp_search(const char *text, const char *pattern, int *prefix, int pattern_len, int *positions, int *count) {
    int text_len = strlen(text);
    int j = 0;
    for (int i = 0; i < text_len; ++i) {
        while (j > 0 && text[i] != pattern[j]) {
            j = prefix[j - 1];
        }
        if (text[i] == pattern[j]) {
            j++;
        }
        if (j == pattern_len) {
            positions[*count] = i - pattern_len + 1;
            *count += 1;
            j = prefix[j - 1];
        }
    }
}

void to_lower(char *str) {
    for (int i = 0; str[i]; ++i) {
        str[i] = tolower(str[i]);
    }
}


void process_string(char *input) {
    // 复制输入字符串
    char text[2048];
    strncpy(text, input, sizeof(text) - 1);
    text[sizeof(text) - 1] = '\0';

    // 转为小写用于匹配
    char lower_text[2048];
    strcpy(lower_text, text);
    to_lower(lower_text);

    int len = strlen(text);

    // 遍历每一个敏感词
    for (int w = 0; w < sensitive_count; ++w) {
        const char *word = sensitive_words[w];
        int word_len = strlen(word);

        int prefix[word_len];
        compute_prefix_function(word, prefix);

        int positions[100];
        int count = 0;
        kmp_search(lower_text, word, prefix, word_len, positions, &count);

        // 对每个匹配到的敏感词，处理其后面的值
        for (int i = 0; i < count; ++i) {
            int pos = positions[i];  // 敏感词在 text 中的起始位置

            // 查找等号或冒号：从敏感词结尾开始向后找
            int start = pos + word_len;
            while (start < len && start < pos + word_len + 10) {  // 限制查找范围
                if (text[start] == '=' || text[start] == ':') {
                    start++;  // 跳过分隔符
                    break;
                }
                start++;
            }
            if (start >= len || text[start - 1] != '=' && text[start - 1] != ':') {
                continue; // 没找到 = 或 :
            }

            // 找到值的起始位置 start
            // 现在确定值的结束位置（遇到逗号、空格、括号、结束符等）
            int end = start;
            while (end < len &&
                   text[end] != ','){ // &&
                //    text[end] != ' ' &&
                //    text[end] != '\t' &&
                //    text[end] != '\n' &&
                //    text[end] != ';' &&
                //    text[end] != ')' &&
                //    text[end] != '}' &&
                //    text[end] != ']') 
                
                end++;
            }

            // 将值部分替换成 *
            for (int j = start; j < end; ++j) {
                text[j] = '*';
            }

            // 可选：避免重复处理重叠的敏感词（比如 password 和 passwd）
            // 这里简单处理，不重复标记即可
        }
    }

    printf("处理后字符串: %s\n", text);
}


int main() {
    char input[] = "password=123456, token=abcdef, phone=12345678901, name=Tom, card=4111111111111111, keysecret=mysecret";
    printf("原始字符串: %s\n", input);
    process_string(input);

    return 0;
}
