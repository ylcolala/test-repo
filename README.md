============================================================
1. PROJECT STRUCTURE
============================================================

src/main/java/com/example/clearing/
├── model/
│   ├── SettlementMessage.java
│   ├── ValidationError.java
│   └── ValidationResult.java
├── validation/
│   ├── MessageValidationRule.java
│   ├── MessageValidationEngine.java
│   ├── ValidationRuleRegistry.java
│   └── ValidationProperties.java
├── rule/
│   ├── TradeIdValidationRule.java
│   ├── CurrencyValidationRule.java
│   └── AmountValidationRule.java
├── config/
│   └── ValidationConfiguration.java
└── service/
    └── SettlementApplicationService.java

src/main/resources/
└── application.yml


============================================================
2. MESSAGE MODEL
============================================================

package com.example.clearing.model;

import java.math.BigDecimal;

public class SettlementMessage {

    private String messageId;
    private String tradeId;
    private String currency;
    private BigDecimal amount;

    public String getMessageId() {
        return messageId;
    }

    public void setMessageId(String messageId) {
        this.messageId = messageId;
    }

    public String getTradeId() {
        return tradeId;
    }

    public void setTradeId(String tradeId) {
        this.tradeId = tradeId;
    }

    public String getCurrency() {
        return currency;
    }

    public void setCurrency(String currency) {
        this.currency = currency;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }
}


============================================================
3. VALIDATION ERROR
============================================================

package com.example.clearing.model;

public record ValidationError(
        String ruleCode,
        String field,
        String errorCode,
        String message
) {}


============================================================
4. VALIDATION RESULT
============================================================

package com.example.clearing.model;

import java.util.List;

public record ValidationResult(
        boolean valid,
        List<ValidationError> errors
) {

    public static ValidationResult success() {
        return new ValidationResult(true, List.of());
    }

    public static ValidationResult failure(
            List<ValidationError> errors) {

        return new ValidationResult(false, errors);
    }
}


============================================================
5. VALIDATION RULE INTERFACE
============================================================

package com.example.clearing.validation;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationError;

import java.util.List;

public interface MessageValidationRule {

    String getRuleCode();

    boolean supports(SettlementMessage message);

    List<ValidationError> validate(
            SettlementMessage message);
}


============================================================
6. TRADE ID VALIDATION RULE
============================================================

package com.example.clearing.rule;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationError;
import com.example.clearing.validation.MessageValidationRule;

import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class TradeIdValidationRule
        implements MessageValidationRule {

    @Override
    public String getRuleCode() {
        return "TRADE_ID_REQUIRED";
    }

    @Override
    public boolean supports(SettlementMessage message) {
        return true;
    }

    @Override
    public List<ValidationError> validate(
            SettlementMessage message) {

        if (message.getTradeId() == null
                || message.getTradeId().isBlank()) {

            return List.of(
                    new ValidationError(
                            getRuleCode(),
                            "tradeId",
                            "TRADE_ID_MISSING",
                            "Trade ID must not be empty"
                    )
            );
        }

        return List.of();
    }
}


============================================================
7. CURRENCY VALIDATION RULE
============================================================

package com.example.clearing.rule;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationError;
import com.example.clearing.validation.MessageValidationRule;

import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Set;

@Component
public class CurrencyValidationRule
        implements MessageValidationRule {

    private static final Set<String> SUPPORTED_CURRENCIES =
            Set.of("USD", "EUR", "GBP", "CNY", "JPY");

    @Override
    public String getRuleCode() {
        return "CURRENCY_SUPPORTED";
    }

    @Override
    public boolean supports(SettlementMessage message) {
        return true;
    }

    @Override
    public List<ValidationError> validate(
            SettlementMessage message) {

        if (message.getCurrency() == null
                || !SUPPORTED_CURRENCIES.contains(
                        message.getCurrency())) {

            return List.of(
                    new ValidationError(
                            getRuleCode(),
                            "currency",
                            "INVALID_CURRENCY",
                            "Unsupported currency"
                    )
            );
        }

        return List.of();
    }
}


============================================================
8. AMOUNT VALIDATION RULE
============================================================

package com.example.clearing.rule;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationError;
import com.example.clearing.validation.MessageValidationRule;

import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.List;

@Component
public class AmountValidationRule
        implements MessageValidationRule {

    @Override
    public String getRuleCode() {
        return "AMOUNT_REQUIRED";
    }

    @Override
    public boolean supports(SettlementMessage message) {
        return true;
    }

    @Override
    public List<ValidationError> validate(
            SettlementMessage message) {

        if (message.getAmount() == null) {

            return List.of(
                    new ValidationError(
                            getRuleCode(),
                            "amount",
                            "AMOUNT_MISSING",
                            "Amount must not be null"
                    )
            );
        }

        return List.of();
    }
}


============================================================
9. YAML CONFIGURATION
============================================================

# application.yml

clearing:
  validation:
    messages:
      settlement:
        rules:
          - code: TRADE_ID_REQUIRED
            enabled: true
            order: 1

          - code: CURRENCY_SUPPORTED
            enabled: true
            order: 2

          - code: AMOUNT_REQUIRED
            enabled: true
            order: 3


============================================================
10. VALIDATION PROPERTIES
============================================================

package com.example.clearing.validation;

import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.List;
import java.util.Map;

@ConfigurationProperties(prefix = "clearing.validation")
public class ValidationProperties {

    private Map<String, MessageConfig> messages;

    public Map<String, MessageConfig> getMessages() {
        return messages;
    }

    public void setMessages(
            Map<String, MessageConfig> messages) {
        this.messages = messages;
    }

    public static class MessageConfig {

        private List<RuleConfig> rules;

        public List<RuleConfig> getRules() {
            return rules;
        }

        public void setRules(List<RuleConfig> rules) {
            this.rules = rules;
        }
    }

    public static class RuleConfig {

        private String code;
        private boolean enabled;
        private int order;

        public String getCode() {
            return code;
        }

        public void setCode(String code) {
            this.code = code;
        }

        public boolean isEnabled() {
            return enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public int getOrder() {
            return order;
        }

        public void setOrder(int order) {
            this.order = order;
        }
    }
}


============================================================
11. VALIDATION CONFIGURATION
============================================================

package com.example.clearing.config;

import com.example.clearing.validation.ValidationProperties;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(ValidationProperties.class)
public class ValidationConfiguration {
}


============================================================
12. VALIDATION RULE REGISTRY
============================================================

package com.example.clearing.validation;

import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Component
public class ValidationRuleRegistry {

    private final Map<String, MessageValidationRule> ruleMap;

    private final ValidationProperties properties;

    public ValidationRuleRegistry(
            List<MessageValidationRule> rules,
            ValidationProperties properties) {

        this.properties = properties;

        this.ruleMap = rules.stream()
                .collect(Collectors.toMap(
                        MessageValidationRule::getRuleCode,
                        Function.identity()
                ));
    }

    public List<MessageValidationRule> getEnabledRules(
            String messageType) {

        var messageConfig =
                properties.getMessages().get(messageType);

        if (messageConfig == null) {
            throw new IllegalArgumentException(
                    "No validation configuration for: "
                            + messageType);
        }

        return messageConfig.getRules().stream()
                .filter(ValidationProperties.RuleConfig::isEnabled)
                .sorted((a, b) ->
                        Integer.compare(a.getOrder(), b.getOrder()))
                .map(config -> {

                    MessageValidationRule rule =
                            ruleMap.get(config.getCode());

                    if (rule == null) {
                        throw new IllegalArgumentException(
                                "Validation rule not found: "
                                        + config.getCode());
                    }

                    return rule;
                })
                .toList();
    }
}


============================================================
13. VALIDATION ENGINE
============================================================

package com.example.clearing.validation;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationError;
import com.example.clearing.model.ValidationResult;

import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class MessageValidationEngine {

    private final ValidationRuleRegistry registry;

    public MessageValidationEngine(
            ValidationRuleRegistry registry) {
        this.registry = registry;
    }

    public ValidationResult validate(
            String messageType,
            SettlementMessage message) {

        List<ValidationError> errors =
                registry.getEnabledRules(messageType)
                        .stream()
                        .filter(rule -> rule.supports(message))
                        .flatMap(rule ->
                                rule.validate(message).stream())
                        .toList();

        if (errors.isEmpty()) {
            return ValidationResult.success();
        }

        return ValidationResult.failure(errors);
    }
}


============================================================
14. VALIDATION EXCEPTION
============================================================

package com.example.clearing.validation;

import com.example.clearing.model.ValidationError;

import java.util.List;

public class MessageValidationException
        extends RuntimeException {

    private final List<ValidationError> errors;

    public MessageValidationException(
            List<ValidationError> errors) {

        super("Message validation failed");

        this.errors = errors;
    }

    public List<ValidationError> getErrors() {
        return errors;
    }
}


============================================================
15. SETTLEMENT APPLICATION SERVICE
============================================================

package com.example.clearing.service;

import com.example.clearing.model.SettlementMessage;
import com.example.clearing.model.ValidationResult;
import com.example.clearing.validation.MessageValidationEngine;
import com.example.clearing.validation.MessageValidationException;

import org.springframework.stereotype.Service;

@Service
public class SettlementApplicationService {

    private final MessageValidationEngine validationEngine;

    public SettlementApplicationService(
            MessageValidationEngine validationEngine) {

        this.validationEngine = validationEngine;
    }

    public void processSettlement(
            SettlementMessage message) {

        // Step 1: Validate incoming message
        ValidationResult result =
                validationEngine.validate(
                        "settlement",
                        message);

        // Step 2: Stop processing if validation fails
        if (!result.valid()) {
            throw new MessageValidationException(
                    result.errors());
        }

        // Step 3: Execute settlement business logic
        executeSettlement(message);
    }

    private void executeSettlement(
            SettlementMessage message) {

        // TODO:
        // Calculate settlement amount
        // Persist settlement result
        // Update settlement ledger
        // Create Outbox event
    }
}


============================================================
16. EXECUTION FLOW
============================================================

Kafka Consumer
      |
      v
XML Parser
      |
      v
XSD Validation
      |
      v
Settlement Message
      |
      v
Message Validation Engine
      |
      v
Validation Rule Registry
      |
      +--------------------------+
      |                          |
      v                          v
TradeIdValidationRule    CurrencyValidationRule
      |                          |
      +------------+-------------+
                   |
                   v
          AmountValidationRule
                   |
                   v
          Aggregate Errors
                   |
          +--------+--------+
          |                 |
          v                 v
       VALID             INVALID
          |                 |
          v                 v
 Settlement Logic       Error Handling
          |                 |
          v                 v
 Database + Outbox      DLQ / Error Store


============================================================
17. HOW TO ADD A NEW RULE
============================================================

For example, add a new rule to validate settlement date.

Step 1: Create a new Java class.

@Component
public class SettlementDateValidationRule
        implements MessageValidationRule {

    @Override
    public String getRuleCode() {
        return "SETTLEMENT_DATE_REQUIRED";
    }

    @Override
    public boolean supports(SettlementMessage message) {
        return true;
    }

    @Override
    public List<ValidationError> validate(
            SettlementMessage message) {

        // Implement settlement date validation

        return List.of();
    }
}

Step 2: Add the rule to application.yml.

- code: SETTLEMENT_DATE_REQUIRED
  enabled: true
  order: 4

Step 3: Restart the application.

The registry automatically discovers the new Spring component.

No changes are required in MessageValidationEngine.


============================================================
18. HOW TO DISABLE A RULE
============================================================

In application.yml:

- code: CURRENCY_SUPPORTED
  enabled: false
  order: 2

The engine will skip this rule.

No Java code changes are required.


============================================================
19. RECOMMENDED PRODUCTION IMPROVEMENTS
============================================================

1. Separate XML syntax validation from business validation.

   XML Parser:
   Checks whether XML is well-formed.

   XSD Validator:
   Checks whether XML conforms to the FpML/SHCH schema.

   Business Validation:
   Checks clearing business rules.

2. Use structured validation errors.

   Each error should contain:
   - messageId
   - businessEventId
   - ruleCode
   - field
   - errorCode
   - errorMessage
   - receivedAt

3. Make validation rules stateless.

   A validation rule should:
   - Validate input data.
   - Return validation errors.
   - Avoid database updates.
   - Avoid Kafka publishing.
   - Avoid modifying the original message.

4. Use sequential Stream processing.

   Avoid parallelStream() because:
   - Rule execution order may matter.
   - Some rules may depend on earlier validation.
   - Error aggregation should be predictable.

5. Handle different error categories separately.

   Malformed XML:
   Permanent error -> DLQ.

   XSD validation failure:
   Permanent error -> DLQ.

   Business validation failure:
   Business exception -> Error Store / Business Exception Queue.

   Database or network failure:
   Transient error -> Retry.

6. Add idempotency.

   Before settlement processing, check the business event ID.

   Use a database unique constraint to prevent duplicate processing.

7. Use the Outbox Pattern.

   Persist settlement result and Outbox event in the same database
   transaction.

   Publish the Outbox event to Kafka asynchronously.

8. Make configuration reloadable if required.

   For runtime rule changes without restarting:
   Use a database or centralized configuration service,
   plus a controlled refresh mechanism.


============================================================
20. RECOMMENDED ARCHITECTURE FOR OTC CLEARING
============================================================

For a real OTC clearing platform, I recommend separating
validation by message type.

For example:

VM Message
    |
    +-- VM XML Validation
    +-- VM XSD Validation
    +-- VM Business Validation
    +-- VM Settlement Processing

PAI Message
    |
    +-- PAI XML Validation
    +-- PAI XSD Validation
    +-- PAI Business Validation
    +-- PAI Settlement Processing

Coupon Message
    |
    +-- Coupon XML Validation
    +-- Coupon XSD Validation
    +-- Coupon Business Validation
    +-- Coupon Settlement Processing

Position Message
    |
    +-- Position XML Validation
    +-- Position XSD Validation
    +-- Position Business Validation
    +-- Position Processing


IMPORTANT DESIGN NOTES:

- The sample above uses a single SettlementMessage type.
- For a production platform, use separate typed validation engines
  and registries for VM, PAI, Coupon and Position.
- Do not mix different message-specific rule types in one registry
  without type-safe dispatch.
- Validate duplicate rule codes and missing rule configuration
  during application startup.
- For dependent rules, use explicit validation phases rather
  than relying only on Stream ordering.
- Ensure permanent validation failures are persisted or sent
  to a DLQ before the original Kafka record is acknowledged.
- Kafka offset is technical identity; businessEventId is the
  business idempotency identity.
