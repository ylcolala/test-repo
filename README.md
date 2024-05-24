# test-repo

public static String escapeXml(String input) {
    StringBuilder sb = new StringBuilder();
    char[] chars = input.toCharArray();

    for (char c : chars) {
        switch (c) {
            case '<':
                sb.append("&lt;");
                break;
            case '>':
                sb.append("&gt;");
                break;
            case '&':
                sb.append("&amp;");
                break;
            case '"':
                sb.append("&quot;");
                break;
            case '\'':
                sb.append("&apos;");
                break;
            default:
                sb.append(c);
        }
    }
    return sb.toString();
}


String userInput = "someUserInput"; // This should come from user input
String escapedInput = escapeXml(userInput);

// Construct the XPath expression with the escaped input
String xpathExpression = "//element[text()='escapedInput']";

// Compile and evaluate the XPath expression
XPath xPath = XPathFactory.newInstance().newXPath();
XPathExpression expr = xPath.compile(xpathExpression.replace("escapedInput", escapedInput));
NodeList nodeList = (NodeList) expr.evaluate(xmlDocument, XPathConstants.NODESET);
