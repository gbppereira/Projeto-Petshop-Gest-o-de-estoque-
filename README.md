import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.FileWriter;
import java.io.IOException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.TimeZone;

import Data.DBContext;

public class MainGUI extends JFrame {
    private List<Racao> estoque;
    private FuncionarioPetShop funcionario;

    private JTextArea textArea;

    public MainGUI() {
        estoque = new ArrayList<>();
        funcionario = new FuncionarioPetShop();

        setTitle("Smart Validity");
        setExtendedState(JFrame.MAXIMIZED_BOTH);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new BorderLayout());

        textArea = new JTextArea();
        JScrollPane scrollPane = new JScrollPane(textArea);

        JButton cadastrarButton = new JButton("Cadastrar Ração");
        cadastrarButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                cadastrarRacao();
            }
        });

        JButton atualizarButton = new JButton("Atualizar Ração");
        atualizarButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                atualizarRacao();
            }
        });

        JButton excluirButton = new JButton("Excluir Ração");
        excluirButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                excluirRacao();
            }
        });

        JPanel buttonPanel = new JPanel();
        buttonPanel.add(cadastrarButton);
        buttonPanel.add(atualizarButton);
        buttonPanel.add(excluirButton);

        add(scrollPane, BorderLayout.CENTER);
        add(buttonPanel, BorderLayout.SOUTH);

        JButton closeButton = new JButton("X");
        closeButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                System.exit(0);
            }
        });
        JPanel closePanel = new JPanel(new FlowLayout(FlowLayout.RIGHT));
        closePanel.add(closeButton);
        add(closePanel, BorderLayout.NORTH);

        JButton salvarCSVButton = new JButton("Salvar em CSV");
        salvarCSVButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                salvarEmCSV();
            }
        });
        buttonPanel.add(salvarCSVButton);
    }

    public void cadastrarRacao() {
        SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
        sdf.setTimeZone(TimeZone.getTimeZone("America/Sao_Paulo"));

        String nomeRacao = JOptionPane.showInputDialog("Nome da ração:");
        String dataValidadeStr = JOptionPane.showInputDialog("Data de validade (dd-MM-yyyy):");
        Date dataValidade;
        try {
            dataValidade = sdf.parse(dataValidadeStr);
        } catch (ParseException e) {
            JOptionPane.showMessageDialog(null, "Data de validade inválida.");
            return;
        }

        double precoRacao;
        try {
            precoRacao = Double.parseDouble(JOptionPane.showInputDialog("Preço da ração:"));
        } catch (NumberFormatException e) {
            JOptionPane.showMessageDialog(null, "Preço inválido.");
            return;
        }

        Racao racao = new Racao(nomeRacao, dataValidade, precoRacao);
        funcionario.cadastrarRacao(estoque, racao);
        updateTextArea();

        // Save to PostgreSQL
        salvarNoPostgreSQL(racao);
    }   

    private void salvarNoPostgreSQL(Racao racao) {
        try {
            DBContext db = new DBContext();
            db.conectarBanco();

            String nome = racao.getNome();
            Date dataValidade = racao.getDataValidade();
            Double preco = racao.getPreco();

            String sql = "INSERT INTO public.racao(nome, data_validade, preco) VALUES ('" +nome + "','" + dataValidade + "','" + preco +"');";
            db.executarUpdateSql(sql);

            db.desconectarBanco();
        } catch (Exception e) {
            e.printStackTrace();
            JOptionPane.showMessageDialog(null, "Erro ao cadastrar ração no PostgreSQL: " + e.getMessage());
        }
    }

    private void atualizarRacao() {
        String nomeAntigo = JOptionPane.showInputDialog("Nome da ração a ser atualizada:");
        Racao racaoAntiga = null;

        for (Racao r : estoque) {
            if (r.getNome().equalsIgnoreCase(nomeAntigo)) {
                racaoAntiga = r;
                break;
            }
        }

        if (racaoAntiga == null) {
            JOptionPane.showMessageDialog(null, "Ração não encontrada.");
            return;
        }

        // Captura as novas informações da ração
        String novoNome = JOptionPane.showInputDialog("Novo nome da ração:");
        racaoAntiga.setNome(novoNome);

        try {
            String novaDataStr = JOptionPane.showInputDialog("Nova data de validade (dd-MM-yyyy):");
            SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
            Date novaData = sdf.parse(novaDataStr);
            racaoAntiga.setDataValidade(novaData);
        } catch (ParseException e) {
            JOptionPane.showMessageDialog(null, "Data de validade inválida.");
            return;
        }

        try {
            double novoPreco = Double.parseDouble(JOptionPane.showInputDialog("Novo preço da ração:"));
            racaoAntiga.setPreco(novoPreco);
        } catch (NumberFormatException e) {
            JOptionPane.showMessageDialog(null, "Preço inválido.");
            return;
        }

        updateTextArea();

        // Atualize no PostgreSQL
        atualizarNoPostgreSQL(racaoAntiga);
    }

    private void atualizarNoPostgreSQL(Racao racao) {
        try {
            DBContext db = new DBContext();
            db.conectarBanco();

            String nomeAntigo = racao.getNome();
            String novoNome = racao.getNome();
            Date novaData = racao.getDataValidade();
            double novoPreco = racao.getPreco();

            String sql = "UPDATE public.racao SET nome = '" + novoNome + "', data_validade = '" + novaData + "', preco = " + novoPreco + " WHERE nome = '" + nomeAntigo + "';";
            db.executarUpdateSql(sql);

            db.desconectarBanco();
        } catch (Exception e) {
            e.printStackTrace();
            JOptionPane.showMessageDialog(null, "Erro ao atualizar ração no PostgreSQL: " + e.getMessage());
        }
    }

    private void excluirRacao() {
        String nomeRacao = JOptionPane.showInputDialog("Nome da ração a ser excluída:");
        Racao racaoParaExcluir = null;

        for (Racao r : estoque) {
            if (r.getNome().equalsIgnoreCase(nomeRacao)) {
                racaoParaExcluir = r;
                break;
            }
        }

        if (racaoParaExcluir == null) {
            JOptionPane.showMessageDialog(null, "Ração não encontrada.");
            return;
        }

        estoque.remove(racaoParaExcluir);
        updateTextArea();

        // Exclua no PostgreSQL
        excluirDoPostgreSQL(racaoParaExcluir);
    }

    private void excluirDoPostgreSQL(Racao racao) {
        try {
            DBContext db = new DBContext();
            db.conectarBanco();

            String nomeRacao = racao.getNome();
            String sql = "DELETE FROM public.racao WHERE nome = '" + nomeRacao + "';";
            db.executarUpdateSql(sql);

            db.desconectarBanco();
        } catch (Exception e) {
            e.printStackTrace();
            JOptionPane.showMessageDialog(null, "Erro ao excluir ração no PostgreSQL: " + e.getMessage());
        }
    }

    public void updateTextArea() {
        textArea.setText("");
        SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
        for (Racao racao : estoque) {
            textArea.append("Nome: " + racao.getNome() + "\n");
            textArea.append("Data de Validade: " + sdf.format(racao.getDataValidade()) + "\n");
            textArea.append("Preço: " + racao.getPreco() + "\n\n");
        }
    }

    private void salvarEmCSV() {
        String caminhoDoArquivo = "C:/Users/z328879/OneDrive - Claro SA/Área de Trabalho/java2/petshop-manager/banco/dados.csv";

        try (FileWriter csvWriter = new FileWriter(caminhoDoArquivo)) {
            csvWriter.append("Nome,Data de Validade,Preço\n");

            SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
            for (Racao racao : estoque) {
                csvWriter.append(escapeSpecialCharacters(racao.getNome())).append(",");
                csvWriter.append(sdf.format(racao.getDataValidade())).append(",");
                csvWriter.append(String.valueOf(racao.getPreco())).append("\n");
            }

            JOptionPane.showMessageDialog(null, "Dados salvos em CSV no caminho: " + caminhoDoArquivo);
        } catch (IOException ex) {
            ex.printStackTrace();
            JOptionPane.showMessageDialog(null, "Erro ao salvar os dados em CSV: " + ex.getMessage());
        }
    }

    private String escapeSpecialCharacters(String input) {
        // Implement your character escaping logic if needed.
        return input;
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(new Runnable() {
            @Override
            public void run() {
                MainGUI mainGUI = new MainGUI();
                mainGUI.setVisible(true);
            }
        });
    }
}

class Racao {
    private String nome;
    private Date dataValidade;
    private double preco;

    public Racao(String nome, Date dataValidade, double preco) {
        this.nome = nome;
        this.dataValidade = dataValidade;
        this.preco = preco;
    }

    public String getNome() {
        return nome;
    }

    public void setNome(String nome) {
        this.nome = nome;
    }

    public Date getDataValidade() {
        return dataValidade;
    }

    public void setDataValidade(Date dataValidade) {
        this.dataValidade = dataValidade;
    }

    public double getPreco() {
        return preco;
    }

    public void setPreco(double preco) {
        this.preco = preco;
    }
}

class FuncionarioPetShop {
    public void cadastrarRacao(List<Racao> estoque, Racao racao) {
        estoque.add(racao);
    }
}
